# AI-Powered Program Management System

> Autonomous weekly reporting pipeline built on n8n — multi-agent orchestration using GPT-4o, Jira, Google Drive, Airtable, and Slack.

**Pilot use case developed as part of the MIT Applied Agentic AI for Organizational Transformation program.**

---

## What it does

Every Monday at 07:00, without manual intervention, the system:

1. Fetches active projects from Airtable
2. Uses GPT-4o to prioritise workstreams and plan the run
3. Queries each Jira project for open sprint status and generates a RAG (Red / Amber / Green) assessment per workstream
4. Detects cross-workstream cascade risks using a dependency map and GPT-4o
5. Processes this week's meeting notes from Google Drive and extracts action items
6. Creates Jira tickets automatically for critical and amber risks and untracked meeting actions
7. Synthesises all data into an exec-ready brief using GPT-4o
8. Posts the brief to a Slack channel with one-click Approve / Request Edit buttons
9. On approval, sends the brief to the exec recipient via Gmail

If any workflow fails at any step, a global error handler posts a structured Slack alert with the failed node name and a direct link to the n8n execution log.

---

## Architecture

```
Weekly Cron (Monday 07:00)
       │
       ▼
Fetch Projects (Airtable)
       │
       ▼
Build Orchestrator Prompt
       │
       ▼
GPT-4o — Prioritise workstreams
       │
       ▼
Parse & validate plan
       │
       ├──────────────────────────────────┐
       ▼                                  │
Call Status Agent (per workstream, parallel)
       │                                  │
       ▼                                  │
Status Records Wrapper                    │
       │                                  │
       ├──────────────┬───────────────────┘
       ▼              ▼
Call Risk Agent   Call Comms Agent
       │              │
       └──────┬───────┘
              ▼
       Combine Results
              │
              ▼
       Call Reporting Agent
              │
              ▼
       Parse for Slack
              │
              ▼
       Post to Slack ──► PM Approves ──► Gmail to Exec
                      └──► PM Requests Edit ──► Slack notification
```

---

## Workflows

The system is composed of 7 n8n workflows. Import them in the order listed.

| File | Workflow name | n8n ID | Purpose |
|---|---|---|---|
| `Error_Notification.json` | Error Notification | `KbsJHQVOmJv0DC48` | Global error handler — attach to all workflows |
| `Status_Agent.json` | Status Agent | `qQYqqVmZX1t34Wdg` | Per-workstream Jira → RAG status analysis |
| `Risk_Agent.json` | Risk Agent | `jQhflq0TQn0v30GM` | Cross-workstream risk detection → Jira tickets |
| `Comms_Agent.json` | Comms Agent | `JFueiBmg3k4Pmazb` | Meeting notes → action items → Jira tasks |
| `Reporting_Agent.json` | Reporting Agent | `54ZepALq88IjywX3` | Synthesise all data → exec brief → Google Drive |
| `Slack_Approval_Handler.json` | Slack Approval Handler | `jJoRC77qtNTny9hW` | PM approve / edit gate → email delivery |
| `Program_Management_Workflow.json` | Program Management Workflow | `SbBBCT5B2KrX7VWS` | Master orchestrator — import last |

---

## Prerequisites

### Accounts and access

- [n8n](https://n8n.io) instance (self-hosted or cloud) — v1.x
- [OpenAI API](https://platform.openai.com) key with access to `gpt-4o` and `gpt-4o-mini`
- [Jira Software Cloud](https://www.atlassian.com/software/jira) — one or more projects with active sprints
- [Airtable](https://airtable.com) account
- [Google Drive](https://drive.google.com) + [Gmail](https://mail.google.com) — same Google account
- [Slack](https://slack.com) workspace with admin access to create an app

### n8n version

Tested on n8n `1.x`. The workflows use:
- `executeWorkflow` node (v1.2 / v1.3)
- `n8n-nodes-base.webhook` (v2.1)
- `n8n-nodes-base.scheduleTrigger` (v1.3)
- `n8n-nodes-base.airtable` (v2.2)
- `n8n-nodes-base.googleDrive` (v3)

---

## Setup

### Step 1 — Airtable

Create a base named `prg_mgmt_base` and add the following two tables.

#### Table: `Projects`

| Field | Type | Notes |
|---|---|---|
| `Name` | Single line text | Workstream / project name |
| `JiraKey` | Single line text | Jira project key (e.g. `PROJ`, `AUTH`) |

#### Table: `Dependency Mapping`

| Field | Type | Notes |
|---|---|---|
| `Upstream` | Single line text | Name of the upstream workstream |
| `Downstream` | Single line text | Name of the downstream workstream |
| `LagDays` | Number | Days of lag between upstream and downstream |
| `DependencyType` | Single select | `blocks` / `feeds` / `informs` |
| `Notes` | Long text | Optional context |

Note the base ID from the Airtable URL (`appXXXXXXXXXXXXXX`) — needed for the `AIRTABLE_BASE_ID` environment variable.

### Step 2 — Google Drive

Create two folders in Google Drive:

| Folder | Purpose |
|---|---|
| `Exec Briefs` (or any name) | Where generated briefs are saved |
| `Meetings` | Drop `.txt` meeting notes here each week |

Note the folder ID for each folder from the URL (`https://drive.google.com/drive/folders/FOLDER_ID`).

After importing the workflows, update:

- **Reporting Agent** → `Create file from text` node → `folderId` → replace with your Exec Briefs folder ID
- **Comms Agent** → `Search files and folders` node → folder ID in both the `queryString` and `filter.folderId` → replace with your Meetings folder ID

### Step 3 — Slack app

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App** → **From scratch**
2. Under **OAuth & Permissions** → **Bot Token Scopes**, add: `chat:write`, `chat:write.public`, `channels:read`
3. Under **Interactivity & Shortcuts** → enable **Interactivity** → set the Request URL to: `https://YOUR_N8N_URL/webhook/slack-approval`
4. Install the app to your workspace
5. Copy the **Bot User OAuth Token** (starts `xoxb-`) for the n8n Slack credential
6. Copy the **Signing Secret** from **Basic Information** → set as `SLACK_SIGNING_SECRET` environment variable
7. Invite the bot to your exec channel: `/invite @your-app-name`

### Step 4 — n8n credentials

Create the following credentials in n8n (**Settings → Credentials → Add credential**).

> The names in the **Name to use** column must match exactly — the imported workflows reference credentials by name.

| Credential type | Name to use | Notes |
|---|---|---|
| OpenAI API | `OpenAi account` | Paste your OpenAI API key |
| Jira Software Cloud API | `Jira SW Cloud account` | Atlassian email + API token from id.atlassian.com/manage-profile/security/api-tokens |
| Airtable Personal Access Token | `Airtable Personal Access Token account` | From airtable.com/create/tokens — scopes: `data.records:read`, `data.records:write`, `schema.bases:read` |
| Google Drive OAuth2 | `Google Drive account` | OAuth2 flow — include Google Drive and Gmail scopes |
| Gmail OAuth2 | `Gmail account` | Same Google account as Drive |
| Slack OAuth2 | `Slack account 2` | OAuth2 flow using your Slack bot token |

### Step 5 — Environment variables

Go to **Settings → Environment Variables** in n8n and add:

| Variable | Description | Example |
|---|---|---|
| `DEBUG` | `true` enables execution logs in all Code nodes. Set to `false` to silence. | `true` |
| `N8N_BASE_URL` | Your n8n instance URL — used in error notification links | `https://your-n8n.example.com` |
| `SLACK_CHANNEL_EXEC` | Slack channel ID for exec brief and error alerts (starts with `C`) | `C0123456789` |
| `SLACK_SIGNING_SECRET` | Slack app signing secret — used for webhook verification | `abc123def456...` |
| `VAR_EXEC_EMAIL_RECIPIENT` | Email address the approved brief is sent to | `exec@company.com` |
| `AIRTABLE_BASE_ID` | Your Airtable base ID | `appR09XcFwtXya371` |

> To find a Slack channel ID: right-click the channel → **View channel details** → scroll to the bottom.

### Step 6 — Import workflows

Import the JSON files into n8n in this order:

1. `Error_Notification.json`
2. `Status_Agent.json`
3. `Risk_Agent.json`
4. `Comms_Agent.json`
5. `Reporting_Agent.json`
6. `Slack_Approval_Handler.json`
7. `Program_Management_Workflow.json`

> **Import order matters.** The sub-workflows must exist before the main workflow is imported, as the `Call 'Status Agent'` and similar nodes reference them by workflow ID.

After import, open `Program_Management_Workflow` and verify that the workflow IDs in each `Call ...` node match the IDs assigned by your n8n instance. If they differ, click each node and re-select the sub-workflow by name from the dropdown.

### Step 7 — Attach the error workflow

For each of the 6 workflows (all except Error Notification itself):

1. Open the workflow
2. Click **⚙ Settings** (top right of the canvas)
3. Set **Error Workflow** to `Error Notification`
4. Save

### Step 8 — Activate

Activate in this order:

1. `Error Notification` — must be active before anything else
2. `Slack Approval Handler` — must be active to receive Slack button clicks
3. `Program Management Workflow` — activates the Monday cron

The four sub-agent workflows (Status, Risk, Comms, Reporting) do not need to be activated — they are invoked by the main workflow.

---

## Running manually

To test without waiting for Monday:

1. Open **Program Management Workflow**
2. Click **Test workflow** (top right)
3. All sub-agents will run in sequence
4. Check each node's Output tab in the execution view

To test a single agent in isolation:

1. Open the agent workflow (e.g. **Status Agent**)
2. Click the trigger node → **Pin test data** → add a sample payload:
   ```json
   { "workstream": "Platform", "jiraKey": "PROJ", "priority": "high" }
   ```
3. Click **Test workflow**

---

## Meeting notes format

The Comms Agent processes `.txt` files dropped into your Google Drive `Meetings` folder. Files can be free text — no strict format is required. The agent uses GPT-4o-mini to extract a meeting summary and action items.

Example:

```
Platform Architecture Review — 2025-04-28

Auth service migration blocked on security review.
Infra team needs Kubernetes upgrade sign-off before sprint end.

Actions:
- Alice to chase security sign-off — PROJECT: AUTH — by Fri
- Bob to raise k8s ticket — PROJECT: INFRA — by EOW
- Carol to review hardening doc — PROJECT: SEC — by EOW
```

Files must have `mimeType='text/plain'` and be modified after the Monday of the current week to be picked up.

---

## Configuration reference

### Jira project keys

Jira keys are read from the `JiraKey` field in your Airtable `Projects` table. The Status Agent validates keys against the pattern `/^[A-Z][A-Z0-9]{1,9}$/` before querying Jira. Keys must match your Jira project keys exactly — case-sensitive.

### RAG status thresholds

The Status Agent uses GPT-4o-mini to assess each workstream:

| Status | Meaning |
|---|---|
| 🟢 `green` | On track — no blockers, velocity stable or improving |
| 🟡 `amber` | At risk — blockers present, velocity declining, or no sprint data |
| 🔴 `red` | Off track — milestone variance > 7 days or critical blocker unresolved > 7 days |

### Risk severity thresholds

The Risk Agent uses GPT-4o to classify risks:

| Severity | Meaning | Jira ticket |
|---|---|---|
| `critical` | Blocks another workstream milestone within 7 days | Yes |
| `amber` | May impact another workstream within 14 days | Yes |
| `green` | Contained — no cascade risk | No |

---

## Troubleshooting

**Slack message not posted**
- Check `SLACK_CHANNEL_EXEC` is a channel ID starting with `C`, not the channel name
- Confirm the bot has been invited to the channel with `/invite`
- Open the **Call Slack** node output and check for `ok: false` and the error field

**Status Agent returns AMBER with `_no_data: true`**
- The Jira project has no open sprint — create a sprint and add at least one ticket
- Check the `JiraKey` in Airtable matches exactly (case-sensitive)

**Comms Agent finds no meeting files**
- Confirm `.txt` files are in the correct Google Drive `Meetings` folder
- Confirm files were modified/created after Monday of the current week
- Check the folder ID in the `Search files and folders` node matches your folder

**OpenAI schema validation errors**
- Validation errors appear in the execution log of the relevant parse node with a raw response preview
- Check your OpenAI API key has sufficient quota
- Status Agent uses `gpt-4o-mini`; all other calls use `gpt-4o`

**Sub-workflow IDs do not match after import**
- Open `Program_Management_Workflow`
- Click each `Call ...` node → re-select the sub-workflow by name from the dropdown

**Enabling debug logs**
- Set `DEBUG=true` in environment variables
- Logs appear under **Output → Logs** in each Code node within an execution

---

## File structure

```
.
├── Program_Management_Workflow.json   # Master orchestrator
├── Status_Agent.json                  # Jira status analysis sub-workflow
├── Risk_Agent.json                    # Risk detection sub-workflow
├── Comms_Agent.json                   # Meeting processing sub-workflow
├── Reporting_Agent.json               # Brief generation sub-workflow
├── Slack_Approval_Handler.json        # PM approval webhook workflow
├── Error_Notification.json            # Global error handler
└── README.md                          # This file
```

---

## Known limitations

- **Slack signature verification not wired**: The HMAC-SHA256 verifier node is implemented but disconnected from the main flow — requires a publicly accessible webhook URL and live Slack connection. See `Slack_Approval_Handler.json` for the implementation.
- **No historical trend tracking**: The `velocity_trend` field is based on the current sprint snapshot only. Historical persistence requires a Status History table in Airtable.
- **No run deduplication**: A duplicate cron trigger or mid-run restart creates a second execution. A deduplication guard is planned for Phase 1.
- **Plain text meeting notes only**: The Comms Agent searches for `mimeType='text/plain'`. Google Docs, PDFs, and other formats are not supported.

---

## Roadmap

| Phase | Focus | Key capabilities |
|---|---|---|
| 1 — Close gaps | Reliability | Status history / trend tracking, PM edit loop, run deduplication guard |
| 2 — Agentic loops | True agency | Self-evaluation & retry, proactive daily RED alerting, Slack Q&A agent |
| 3 — Autonomous planning | Initiative | Sprint planning agent, dynamic dependency cascading, stakeholder-personalised briefs |
| 4 — Multi-agent | Collaboration | Agent-to-agent delegation, finance / HR / CI-CD data agents, GPT-4o tool use |

---

## Contributing

Issues and pull requests are welcome. When contributing:

- Keep each sub-workflow self-contained with its own error handling
- All Code nodes must use the DEBUG-gated log helper: `const log = (...args) => { if ($env.DEBUG === 'true') console.log(...args); };`
- Validate all OpenAI JSON responses before passing data downstream
- Test with `DEBUG=true` and pinned test data before submitting

---

## License

MIT — see `LICENSE` for details.

---

## Acknowledgements

Built as a capstone pilot for the [MIT Applied Agentic AI for Organizational Transformation](https://professional.mit.edu) program.

Tools: [n8n](https://n8n.io) · [OpenAI](https://openai.com) · [Jira](https://www.atlassian.com/software/jira) · [Airtable](https://airtable.com) · [Google Drive](https://drive.google.com) · [Slack](https://slack.com)
