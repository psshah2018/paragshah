# Slack setup

This document covers the complete Slack app configuration required to run the system. Two separate Slack integrations are involved: the outbound message posting (used to deliver the exec brief and error alerts) and the inbound webhook (used to receive PM button clicks).

---

## Overview

| Integration | Direction | Used by | Purpose |
|---|---|---|---|
| Slack OAuth2 Bot | Outbound | Orchestrator, Error Notification, Slack Approval Handler | Post messages to the exec channel |
| Slack Interactivity Webhook | Inbound | Slack Approval Handler | Receive Approve / Edit button payloads |

Both are configured through a single Slack app.

---

## Part 1 — Create the Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App**
3. Choose **From scratch**
4. Enter:
   - **App Name:** `PM Agent` (or any name — this appears as the message sender)
   - **Pick a workspace:** select your workspace
5. Click **Create App**

---

## Part 2 — Configure OAuth scopes

1. In the left sidebar, click **OAuth & Permissions**
2. Scroll to **Scopes** → **Bot Token Scopes**
3. Click **Add an OAuth Scope** and add each of the following:

| Scope | Purpose |
|---|---|
| `chat:write` | Post messages to channels the bot has been invited to |
| `chat:write.public` | Post messages to public channels without being invited |
| `channels:read` | Read channel information (used for channel ID resolution) |

4. Scroll up and click **Install to Workspace**
5. Review the permissions and click **Allow**
6. Copy the **Bot User OAuth Token** — it starts with `xoxb-`

> Keep this token secure. It is the credential used by n8n to post messages.

---

## Part 3 — Configure interactivity

The Slack Approval Handler receives a POST request from Slack whenever the PM clicks **Approve and send** or **Request edit** on the brief message. Slack requires an HTTPS endpoint to send these payloads to.

1. In the left sidebar, click **Interactivity & Shortcuts**
2. Toggle **Interactivity** to **On**
3. Set the **Request URL** to your n8n webhook URL:

```
https://YOUR_N8N_URL/webhook/slack-approval
```

Replace `YOUR_N8N_URL` with your n8n instance domain. The path `/webhook/slack-approval` is hardcoded in the Slack Approval Handler webhook node.

4. Click **Save Changes**

> The Request URL must be publicly accessible over HTTPS. Slack validates the URL at save time by sending a challenge request. If you are running n8n locally, use a tunnel (see [Tunnelling for local development](#tunnelling-for-local-development) below).

---

## Part 4 — Copy the signing secret

The Slack signing secret is used by the signature verifier node to authenticate that inbound webhook payloads genuinely come from Slack.

1. In the left sidebar, click **Basic Information**
2. Scroll to **App Credentials**
3. Click **Show** next to **Signing Secret**
4. Copy the value

Set this as the `SLACK_SIGNING_SECRET` environment variable in n8n.

---

## Part 5 — Set up n8n credentials

In n8n, go to **Settings → Credentials → Add credential**.

### Slack OAuth2 credential

1. Select credential type: **Slack OAuth2 API**
2. Name it exactly: `Slack account 2`

> The credential name must match exactly — all Slack nodes in the workflows reference credentials by this name.

3. Paste your **Bot User OAuth Token** (`xoxb-...`) into the Access Token field
4. Save

### Test the credential

1. Open the **Error Notification** workflow
2. Click **Send a message** (Slack node)
3. Click **Test step** — if the credential is working, you will see a success response from Slack

---

## Part 6 — Create and configure the exec channel

1. In Slack, create a channel (or use an existing one) for exec brief delivery — e.g. `#program-briefs`
2. Invite the PM Agent bot: type `/invite @PM Agent` in the channel
3. Find the channel ID:
   - Right-click the channel name in the sidebar
   - Click **View channel details**
   - Scroll to the bottom — the channel ID is shown (format: `C0123456789`)
4. Set this as the `SLACK_CHANNEL_EXEC` environment variable in n8n

> Use the channel **ID**, not the channel name. The `Call Slack` node and all Slack nodes reference the channel by ID. Names can change; IDs cannot.

---

## Part 7 — Activate the Slack Approval Handler

The Slack Approval Handler must be **active** in n8n to receive button payloads from Slack. Unlike the sub-agent workflows, this one is not called by the orchestrator — it runs independently in response to Slack events.

1. Open **Slack Approval Handler** workflow
2. Toggle the workflow to **Active** (top right switch)
3. Confirm the webhook URL is live by checking **Webhook URLs** in the workflow node settings — the production URL (not the test URL) should be shown

---

## How the brief message works

When the orchestrator posts the exec brief to Slack, it uses the **Slack Block Kit** format. The message contains:

```
┌─────────────────────────────────────────────┐
│  Weekly exec brief ready for review          │
├──────────────┬──────────────────────────────┤
│ Overall status  │ Generated                  │
│ AMBER           │ Mon Apr 28 2025            │
├─────────────────────────────────────────────┤
│ [Brief preview — first 2900 characters]      │
├─────────────────────────────────────────────┤
│ Open full brief in Google Drive →            │
├─────────────────────────────────────────────┤
│  [Approve and send]    [Request edit]        │
└─────────────────────────────────────────────┘
```

The **Approve and send** button has `action_id: approve_brief` and carries the Google Drive URL as its value.

The **Request edit** button has `action_id: request_edit`.

When clicked, Slack sends a POST request to the Slack Approval Handler webhook with the full interaction payload including the action ID, the PM's Slack username, and the button value.

---

## Approval flow

```
PM clicks "Approve and send"
  → Slack POSTs to /webhook/slack-approval
  → Payload parser extracts actionId = "approve_brief"
  → IF routes to approve branch
  → Gmail sends brief to VAR_EXEC_EMAIL_RECIPIENT
  → Slack posts confirmation to exec channel
```

```
PM clicks "Request edit"
  → Slack POSTs to /webhook/slack-approval
  → Payload parser extracts actionId = "request_edit"
  → IF routes to edit branch
  → Slack posts "Edit Requested by PM" to exec channel
```

---

## Signature verification (pending production wiring)

The Slack Approval Handler includes a full HMAC-SHA256 signature verifier (`Code in JavaScript1`) that is implemented but not yet connected to the main flow. It is designed to sit between the Webhook node and the payload parser.

When wired, the verification sequence is:

1. Extract `x-slack-signature` and `x-slack-request-timestamp` from request headers
2. Reject if either header is missing
3. Reject if the timestamp is more than 300 seconds old (replay attack protection)
4. Compute `HMAC-SHA256(SLACK_SIGNING_SECRET, "v0:{timestamp}:{rawBody}")`
5. Compare using `crypto.timingSafeEqual()` — constant-time comparison prevents timing attacks
6. Reject with 401 if signatures do not match
7. Respond with 200 immediately after verification to satisfy Slack's 3-second timeout

To wire the verifier when deploying to production:

1. Delete the wire: `Webhook → Code in JavaScript` (payload parser)
2. Add wire: `Webhook → Code in JavaScript1` (verifier)
3. Add wire: `Code in JavaScript1 → Respond to Webhook` (200 response)
4. Add wire: `Respond to Webhook → Code in JavaScript` (payload parser)

---

## Tunnelling for local development

If you are running n8n locally and need Slack to reach your webhook, use a tunnel to expose your local port publicly.

### Using ngrok

```bash
# Install ngrok: https://ngrok.com/download
ngrok http 5678
```

Copy the HTTPS URL (e.g. `https://abc123.ngrok.io`) and use it as your Slack Interactivity Request URL:

```
https://abc123.ngrok.io/webhook/slack-approval
```

> ngrok URLs change on each restart. Update the Slack app's Request URL each time you restart the tunnel.

### Using n8n's built-in tunnel

```bash
n8n start --tunnel
```

n8n will print a public tunnel URL. Use this as your Slack Request URL. Note that the built-in tunnel is for development only and should not be used in production.

---

## Troubleshooting

**Slack posts are not appearing in the channel**

- Check that the bot has been invited to the channel with `/invite @PM Agent`
- Verify `SLACK_CHANNEL_EXEC` is a channel ID starting with `C`, not the channel name
- Open the `Call Slack` node output in an n8n execution and check for `ok: false` and the `error` field

**Button clicks are not reaching n8n**

- Confirm the Slack Approval Handler workflow is Active in n8n
- Confirm the Interactivity Request URL in the Slack app matches your n8n webhook URL exactly
- Check n8n executions — a new execution should appear when a button is clicked
- If running locally, confirm your tunnel is active and the URL is current

**`invalid_auth` error from Slack API**

- The Bot User OAuth Token has expired or been revoked
- Reinstall the app to your workspace under **OAuth & Permissions → Install to Workspace**
- Update the n8n `Slack account 2` credential with the new token

**Payload parser receives empty body**

- Slack sends payloads as `application/x-www-form-urlencoded`, not JSON
- The payload is URL-encoded under the `payload` key and must be decoded with `decodeURIComponent()`
- This is already handled in the `Code in JavaScript` payload parser node — if you see empty body errors, check that the Webhook node has `Raw Body` enabled in its settings

**Signature verification errors (when verifier is wired)**

- Confirm `SLACK_SIGNING_SECRET` is set correctly in n8n environment variables
- The raw body must be captured before any parsing — ensure the Webhook node has `Include Headers` and `Raw Body` enabled
- Timestamp errors (`Request is Xs old`) indicate the n8n server clock is out of sync — check system time
