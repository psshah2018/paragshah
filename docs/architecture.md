# Architecture

This document describes the system design, data flow, agent responsibilities, and node-level detail for the AI Program Management System.

---

## Table of contents

- [System overview](#system-overview)
- [Design principles](#design-principles)
- [Workflow inventory](#workflow-inventory)
- [Data flow](#data-flow)
- [Workflow detail](#workflow-detail)
  - [Orchestrator](#1-orchestrator)
  - [Status Agent](#2-status-agent)
  - [Risk Agent](#3-risk-agent)
  - [Comms Agent](#4-comms-agent)
  - [Reporting Agent](#5-reporting-agent)
  - [Slack Approval Handler](#6-slack-approval-handler)
  - [Error Notification](#7-error-notification)
- [Data schemas](#data-schemas)
- [External integrations](#external-integrations)
- [Security](#security)
- [Error handling strategy](#error-handling-strategy)

---

## System overview

The system is a multi-agent orchestration pipeline that automates the weekly program management reporting cycle. A master orchestrator workflow triggers every Monday, fans out to three parallel specialist agents, collects their outputs, and produces an exec-ready brief that is routed through a human approval gate before delivery.

```
                        ┌─────────────────────────────────┐
                        │         Airtable                 │
                        │  Projects · Dependency Mapping   │
                        └──────────────┬──────────────────┘
                                       │
                              ┌────────▼────────┐
                              │   Orchestrator   │  ← Monday 07:00 cron
                              │   (GPT-4o plan) │
                              └────────┬────────┘
                                       │ fan-out (parallel)
               ┌───────────────────────┼───────────────────────┐
               ▼                       ▼                       ▼
      ┌────────────────┐    ┌─────────────────┐    ┌──────────────────┐
      │  Status Agent  │    │   Risk Agent     │    │   Comms Agent    │
      │  Jira → RAG    │    │  Dependencies +  │    │  Drive notes →   │
      │  per workstream│    │  GPT-4o → risks  │    │  GPT-4o-mini →   │
      │  (GPT-4o-mini) │    │  → Jira tickets  │    │  actions → Jira  │
      └───────┬────────┘    └────────┬─────────┘    └────────┬─────────┘
              │                      │                        │
              └──────────────────────┴────────────────────────┘
                                     │ merge
                            ┌────────▼────────┐
                            │ Reporting Agent  │
                            │ GPT-4o → brief  │
                            │ → Google Drive  │
                            └────────┬────────┘
                                     │
                            ┌────────▼────────┐
                            │  Slack post      │
                            │  Approve / Edit  │
                            └────────┬────────┘
                                     │ on approve
                            ┌────────▼────────┐
                            │  Gmail → Exec    │
                            └─────────────────┘

        ┌─────────────────────────────────────────┐
        │  Error Notification (cross-cutting)      │
        │  Fires on any failure in any workflow    │
        └─────────────────────────────────────────┘
```

---

## Design principles

**Sub-workflow pattern.** Each agent is an independent n8n workflow invoked by the orchestrator via `executeWorkflow` nodes. This means agents can be tested, debugged, and updated in isolation without touching the main pipeline.

**Fail-safe defaults.** Every agent returns a safe fallback rather than crashing the pipeline on missing data. The Status Agent returns AMBER with `_no_data: true` when a sprint is empty. The Reporting Agent uses explicit "no data — do not invent" instructions when any input section is absent.

**Schema validation at every boundary.** All OpenAI JSON responses are parsed and validated against required field schemas before data passes downstream. Malformed responses throw descriptive errors rather than propagating nulls silently.

**DEBUG-gated logging.** All Code nodes use a `log()` helper gated on the `DEBUG` environment variable. Logging is off by default in production and can be enabled per-execution without code changes.

**Idempotent Jira creation.** Both the Risk Agent and Comms Agent check for duplicate Jira issues before creating new ones, preventing duplicate tickets on re-runs.

---

## Workflow inventory

| File | Role | Trigger | Active |
|---|---|---|---|
| `workflows/core/01_orchestrator.json` | Master pipeline coordinator | Schedule (Monday 07:00) | Yes |
| `workflows/agents/02_status_agent.json` | Per-workstream RAG status | Called by orchestrator | No — sub-workflow |
| `workflows/agents/03_risk_agent.json` | Cross-workstream risk detection | Called by orchestrator | No — sub-workflow |
| `workflows/agents/04_comms_agent.json` | Meeting notes processing | Called by orchestrator | No — sub-workflow |
| `workflows/output/05_reporting_agent.json` | Exec brief generation | Called by orchestrator | No — sub-workflow |
| `workflows/output/06_slack_approval_handler.json` | PM approval gate | Webhook (Slack button) | Yes |
| `workflows/system/07_error_notification.json` | Global error handler | Error trigger | Yes |

---

## Data flow

### Phase 1 — Context loading

The orchestrator fetches all active project records from Airtable and builds a prioritised workstream plan using GPT-4o. The LLM output is validated — if it is malformed or empty, the system falls back to the full Airtable project list rather than aborting.

```
Airtable (Projects table)
  → Build prompt
  → GPT-4o
  → [{ workstream, jiraKey, priority }] per workstream
```

### Phase 2 — Parallel agent execution

The orchestrator fans out to three agents simultaneously. Each agent receives the same `statusRecords` array (once it is available from the Status Agent) plus agent-specific inputs.

```
Per workstream (parallel):
  Status Agent ← { workstream, jiraKey, priority }
  → { rag_status, days_variance, blockers, velocity_trend, summary }

Status Records Wrapper collects all per-workstream results, then fans out:

  Risk Agent  ← { statusRecords, dependencyMap }
  → { overall_program_rag, risks[], summary }

  Comms Agent ← { statusRecords }
  → { commsSummary: { summary, action_items[] } }
```

### Phase 3 — Synthesis and delivery

The Combine node waits for all three agent outputs, then the Reporting Agent synthesises them into a Markdown brief.

```
Combine ← status + risk + comms
  → Reporting Agent
  → GPT-4o (gpt-4o, 2048 tokens, temp 0.3)
  → Markdown brief
  → Google Drive (saved as doc)
  → Slack (posted with Approve / Edit buttons)
  → [on approve] Gmail → exec recipient
```

---

## Workflow detail

### 1. Orchestrator

**File:** `workflows/core/01_orchestrator.json`  
**Trigger:** Schedule — every Monday at 07:00  
**Purpose:** Owns the weekly pipeline. Validates project data, plans the run, fans out to agents, collects results, and delivers the brief to Slack.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| Weekly cron | Schedule Trigger | Fires the pipeline every Monday at 07:00 |
| Fetch program context | Airtable | Reads all records from the `Projects` table |
| If | IF | Guards against empty Airtable result — aborts with a descriptive error if no projects found |
| No Projects Found | Code | Throws a descriptive error listing three diagnostic checks (base ID, table contents, token permissions) |
| Build Orchestrator prompt | Code | Validates records (filters any missing Name or JiraKey), builds the LLM system prompt with the active project list |
| Orchestrator OpenAI API call | HTTP Request | Calls `gpt-4o` — returns a prioritised JSON array of workstreams |
| Parse orchestrator plan | Code | Parses and validates LLM output. Falls back to the full Airtable list if JSON is malformed, empty, or more than 50% invalid |
| Call 'Status Agent' | Execute Workflow | Invokes Status Agent once per workstream in parallel (`mode: each`, `waitForSubWorkflow: true`) |
| Status Records Wrapper | Code | Collects all per-workstream status results into a single array, stamps each with `weekStart` |
| Call 'Risk Agent' | Execute Workflow | Passes `statusRecords` to Risk Agent |
| Call Comms Agent | Execute Workflow | Passes `statusRecords` to Comms Agent |
| Combine Results of All 3 Agents | Merge | Waits for all three agent outputs using `combineByPosition` with `includeUnpaired: true` |
| Call Reporting Agent | Execute Workflow | Passes merged `{ statusRecords, riskSummary, commsSummary }` to Reporting Agent |
| Parse for Slack Call | Code | Builds Slack Block Kit payload with RAG badge, brief preview (capped at 2900 chars), Drive link, and Approve / Edit action buttons |
| Call Slack | HTTP Request | Posts the Block Kit message to the exec channel via `chat.postMessage`. On error: continues (does not fail the run) |
| Check Slack Response | Code | Logs `ok: false` as a warning — a Slack failure does not mark the weekly run as failed |

---

### 2. Status Agent

**File:** `workflows/agents/02_status_agent.json`  
**Trigger:** Called by Orchestrator (Execute Workflow Trigger)  
**Input:** `{ workstream, jiraKey, priority }`  
**Output:** `{ rag_status, days_variance, blockers[], velocity_trend, summary, workstream, jiraKey, as_of }`  
**Purpose:** Queries the Jira open sprint for a single workstream and produces a structured RAG status record using GPT-4o-mini.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| Execute Workflow Trigger | Workflow Trigger | Receives `{ workstream, jiraKey, priority }` from the orchestrator |
| Validate Input | Code | Validates `jiraKey` against pattern `/^[A-Z][A-Z0-9]{1,9}$/`. Validates `workstream` is non-empty. Warns on unexpected priority values. Strips trailing hyphens from keys. |
| Get many issues | Jira | JQL: `project = {jiraKey} AND sprint in openSprints() ORDER BY updated DESC` — returns all tickets in the open sprint |
| If | IF | Checks whether the Jira query returned at least one ticket |
| No Sprint Data | Code | Returns a default AMBER record with `_no_data: true` when no sprint exists — bypasses OpenAI entirely |
| Agregate tickets | Code | Collects all Jira ticket items into a flat array alongside the workstream name. Logs ticket count and warns if zero tickets reach this node despite the sprint check passing. |
| Program Status Analysis OpenAI API call | HTTP Request | Calls `gpt-4o-mini` (temp 0.1, 1024 tokens, 60s timeout, retry: 3 × 5s). Returns structured JSON: `{ rag_status, days_variance, blockers[], velocity_trend, summary }` |
| Parse Result | Code | Validates all required fields are present. Validates `rag_status` is `red`/`amber`/`green`. Validates `blockers` is an array. Enriches with `workstream`, `jiraKey`, `as_of` timestamp. |

#### RAG assessment logic

The Status Agent prompt instructs GPT-4o-mini to assess each workstream using these criteria:

| Status | Conditions |
|---|---|
| `green` | No blockers, velocity stable or improving, no milestone variance |
| `amber` | Blockers present, velocity declining, or no sprint data (`_no_data: true`) |
| `red` | Days variance > 7, or critical blocker unresolved for 7+ days |

---

### 3. Risk Agent

**File:** `workflows/agents/03_risk_agent.json`  
**Trigger:** Called by Orchestrator (Execute Workflow Trigger)  
**Input:** `{ statusRecords[] }`  
**Output:** `{ riskSummary: [{ overall_program_rag, risks[], summary }] }`  
**Purpose:** Cross-workstream risk detection using a dependency map from Airtable. Creates Jira tickets for critical and amber risks. Returns a full risk summary for inclusion in the exec brief.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| Execute Workflow Trigger | Workflow Trigger | Receives `{ statusRecords }` from the orchestrator |
| Code — wrap trigger input | Code | Extracts and validates `statusRecords` array from the trigger payload. Handles both string (JSON-serialised) and array input types. |
| Check Project Dependencies | Airtable | Reads the `Dependency Mapping` table — upstream/downstream workstream pairs with lag days, dependency type, and notes |
| Assemble Status and Dependencies | Code | Combines `statusRecords` with the dependency map. Builds `dependencyContext` string — either serialised JSON or a fallback "no dependency data" instruction. Warns if dependency map is empty. |
| Risk Detection OpenAI API Call | HTTP Request | Calls `gpt-4o` (temp 0.1, retry: 3 × 5s). Returns `{ overall_program_rag, risks[], summary }`. Each risk includes: `id, workstream, jiraKey, description, severity, cascade_targets[], days_until_impact, owner, blocker_age_days, recommended_action` |
| Parse Risk Analysis Response | Code | Validates `risks` is a non-empty array. Validates each risk has `severity`, `description`, and `owner`. Validates severity against `['critical', 'amber', 'green']`. |
| If | IF | Checks `Array.isArray($json.risks) && $json.risks.some(r => r.severity === 'critical' \|\| r.severity === 'amber')`. Routes true → Jira creation loop. Also fires `Pass Risk Summary` simultaneously on slot 0. |
| Splits Critical Risks into Separate Items | Code | Filters risks for `severity === 'critical'` or `severity === 'amber'`. Emits one item per risk for the Jira loop. |
| Pass Risk Summary to Main Workflow | Code | Wraps the full risks array (all severities) as `[{ json: { riskSummary } }]` for the orchestrator's Combine node |
| Loop Over Items | Split in Batches | Iterates one risk at a time through the Jira deduplication and creation chain |
| Get Current Issues from Jira | Jira | Fetches existing open Jira issues to support duplicate detection |
| Check for Duplicates | IF | Skips Jira creation if a matching issue already exists |
| Get Project Id | HTTP Request | Resolves Jira project ID from the key |
| Get WorkTypes for Project | HTTP Request | Fetches available work types for the project |
| Get Id of Risk issueType | Code | Finds the internal ID for the `Risk` issue type |
| Create an issue | Jira | Creates a Jira risk ticket with description, severity, and recommended action |

#### Risk severity thresholds

| Severity | Meaning | Jira ticket |
|---|---|---|
| `critical` | Blocks another workstream milestone within 7 days | Yes |
| `amber` | May impact another workstream within 14 days | Yes |
| `green` | Contained — no cascade risk | No |

The Risk Agent prompt rules:
- A blocker aged 7+ days with no owner response is always `critical`
- Declining velocity on a workstream that feeds another is `amber` minimum
- Do not invent risks not supported by the data

---

### 4. Comms Agent

**File:** `workflows/agents/04_comms_agent.json`  
**Trigger:** Called by Orchestrator (Execute Workflow Trigger)  
**Input:** `{ statusRecords[] }`  
**Output:** `{ commsSummary: { summary, action_items[], meeting_name, decisions[] } }`  
**Purpose:** Processes `.txt` meeting notes from a designated Google Drive folder. Extracts action items, creates Jira tasks for each, and returns a meeting digest for inclusion in the exec brief.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| Execute Workflow Trigger | Workflow Trigger | Receives `{ statusRecords }` — uses `statusRecords[0].weekStart` for the Drive search date filter |
| Search files and folders | Google Drive | Searches the `Meetings` folder for `.txt` files with `modifiedTime > weekStart`. Constrained by folder ID and MIME type. |
| If | IF | Checks whether the Drive search returned any files. Routes false → Return to Main Workflow with an empty summary. |
| Download file | Google Drive | Downloads binary content of each meeting file |
| Extract Text from Binary File | Code | Uses `this.helpers.getBinaryDataBuffer()` to read file binary, then `.toString('utf-8')`. Validates MIME type is text. Caps text at 8000 characters. |
| Meeting Analysis OpenAI API Call | HTTP Request | Calls `gpt-4o-mini` (retry: 3 × 5s). Returns structured JSON: `{ meeting_name, summary, action_items[], decisions[] }`. Each action item: `{ description, owner, due_date, project }` |
| Parse OpenAI Response and Prepare Action Items | Code | Validates `summary` is a non-empty string. Validates `action_items` is an array. Marks incomplete items with `_skip: true` (missing `description` or `owner`). Filters skipped items before Jira creation. |
| Check for Action Items | IF | Routes true (action items found) → Jira loop. Routes false (no action items) → Return to Main Workflow. |
| Loop Over Items | Split in Batches | Iterates one action item at a time through the Jira creation chain |
| Get many issues | Jira | Fetches existing Jira issues for duplicate detection |
| Check for Duplicates | IF | Skips Jira creation if a matching task already exists |
| Get Project Id | HTTP Request | Resolves Jira project ID |
| Get WorkTypes for Project | HTTP Request | Fetches available work types |
| Get Id of Task issueType | Code | Finds the internal ID for the `Task` issue type |
| Create Jira issue(s) | Jira | Creates a Jira task with title, owner, and priority from the action item |
| Return to Main Workflow | Code | Reads the OpenAI response via `$('Meeting Analysis OpenAI API Call').first().json` (named reference, not `$input` — works regardless of where the node sits in the flow). Returns `[{ json: { commsSummary: parsed } }]`. |

---

### 5. Reporting Agent

**File:** `workflows/output/05_reporting_agent.json`  
**Trigger:** Called by Orchestrator (Execute Workflow Trigger)  
**Input:** `{ statusRecords[], riskSummary[], commsSummary{} }`  
**Output:** `{ briefMarkdown, overallRag, docUrl, fileId, driveSuccess, wordCount, generatedAt }`  
**Purpose:** Synthesises status, risk, and communications data into an exec-ready Markdown brief, saves it to Google Drive, and returns the full package to the orchestrator for Slack delivery.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| When Executed by Another Workflow | Workflow Trigger | Receives merged `{ statusRecords, riskSummary, commsSummary }` from the orchestrator's Combine node |
| Reporting Agent OpenAI Call | HTTP Request | Calls `gpt-4o` (temp 0.3, 2048 tokens, retry: 3 × 5s, maxTries: 3). Each empty data section includes explicit "do not invent" instructions. Returns Markdown. |
| Parse OpenAI Reporting Response | Code | Validates brief is non-empty and at least 100 words. Detects overall RAG status from the first 300 characters. Warns (does not throw) if expected section markers (`##`, `RAG`, `risk`) are absent. |
| Create file from text | Google Drive | Saves the Markdown brief to the configured Exec Briefs folder. `onError: continueRegularOutput` — Drive failure does not abort the pipeline. |
| Drive Result Handler | Code | Per-item mode. Normalises Drive upload result into `{ docUrl, fileId, driveSuccess }`. Returns a fallback `https://drive.google.com` URL if the upload failed. |
| Merge | Merge | Combines slot 0 (Drive Result Handler) and slot 1 (brief markdown) using `combineByPosition` with `includeUnpaired: true`. Ensures the merge always completes even if Drive upload failed. |
| Return Merged Data to Main Workflow | Code | Assembles the final package from the merged item: `docUrl`/`fileId` from Drive, and `briefMarkdown`/`overallRag`/`wordCount`/`generatedAt` from the parse node. |

#### Brief structure (GPT-4o system prompt)

The Reporting Agent instructs GPT-4o to produce exactly five sections:

1. Overall status (RAG badge: `RED` / `AMBER` / `GREEN`)
2. Workstream summaries (one paragraph per workstream)
3. Top risks (bullet list — severity, description, recommended action)
4. Decisions needed (bullet list of items requiring exec input)
5. Key actions from meetings (bullet list with owners)

Target length: 350–450 words. Tone: direct, factual, no filler.

---

### 6. Slack Approval Handler

**File:** `workflows/output/06_slack_approval_handler.json`  
**Trigger:** Webhook — POST to `/webhook/slack-approval`  
**Purpose:** Receives Slack interactive component payloads when the PM clicks Approve or Request Edit on the brief message. Routes to email delivery on approval, or posts an edit notification on rejection.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| Webhook | Webhook | Listens on POST `/webhook/slack-approval`. `onError: continueRegularOutput` to prevent payload drops. |
| Code in JavaScript (payload parser) | Code | URL-decodes and JSON-parses the Slack `payload` field. Extracts `actionId`, `actionVal`, `userName`, `approved` flag, `docUrl`, and `clickedAt`. |
| If | IF | Strict string comparison on `actionId`. `approve_brief` → true branch. Any other action → false branch. |
| Send a message (Gmail) | Gmail | On approval: sends the brief email to `VAR_EXEC_EMAIL_RECIPIENT` with date and Drive link |
| Send a message1 (Slack) | Slack | On approval: posts confirmation to exec channel |
| Send a message2 (Slack) | Slack | On edit request: posts "Edit Requested by PM" notification to exec channel |
| Code in JavaScript1 (signature verifier) | Code | HMAC-SHA256 signature verification using `SLACK_SIGNING_SECRET`. Includes 5-minute replay attack window and constant-time comparison via `crypto.timingSafeEqual()`. **Currently disconnected from main flow — pending production wiring.** |
| Rejection Path - 401 | Respond to Webhook | Returns HTTP 401 on auth failure (used by verifier when wired) |
| Respond to Webhook | Respond to Webhook | Returns HTTP 200 to satisfy Slack's 3-second response requirement |

---

### 7. Error Notification

**File:** `workflows/system/07_error_notification.json`  
**Trigger:** Error Trigger — fires automatically when any attached workflow fails  
**Purpose:** Global error handler. Extracts structured error information and posts a Slack alert with the workflow name, failed node, error type, error message, and a direct link to the n8n execution log.

#### Node map

| Node | Type | Purpose |
|---|---|---|
| On Workflow Error | Error Trigger | Fires when any workflow that has this set as its `errorWorkflow` encounters an unhandled exception |
| Build Error Message | Code | Extracts `workflow.name`, `execution.lastNodeExecuted`, `execution.error.message`, `execution.error.name`, and `execution.id`. Builds a formatted Slack markdown message. Constructs execution URL from `N8N_BASE_URL`. |
| Send a message | Slack | Posts the error alert to `SLACK_CHANNEL_EXEC` with `mrkdwn: true` for formatted output |

#### Attachment

This workflow must be set as the `errorWorkflow` in the settings of all six other workflows. It is the only workflow that does not have an error workflow of its own.

---

## Data schemas

### Status record (output of Status Agent)

```json
{
  "rag_status":      "red | amber | green",
  "days_variance":   0,
  "blockers": [
    { "description": "string", "owner": "string", "age_days": 0 }
  ],
  "velocity_trend":  "stable | improving | declining",
  "summary":         "2 sentence plain English summary",
  "workstream":      "string",
  "jiraKey":         "string",
  "as_of":           "ISO 8601 timestamp",
  "_no_data":        false
}
```

### Risk record (output of Risk Agent)

```json
{
  "overall_program_rag": "red | amber | green",
  "risks": [
    {
      "id":                 "RISK-001",
      "workstream":         "string",
      "jiraKey":            "string",
      "description":        "one sentence",
      "severity":           "critical | amber | green",
      "cascade_targets":    ["workstream names"],
      "days_until_impact":  0,
      "owner":              "string",
      "blocker_age_days":   0,
      "recommended_action": "one sentence"
    }
  ],
  "summary": "2–3 sentence plain English summary"
}
```

### Comms summary (output of Comms Agent)

```json
{
  "meeting_name": "string",
  "summary":      "string",
  "action_items": [
    {
      "description": "string",
      "owner":       "string",
      "due_date":    "YYYY-MM-DD",
      "project":     "JIRA-KEY"
    }
  ],
  "decisions": ["string"]
}
```

### Orchestrator plan (output of GPT-4o in Orchestrator)

```json
[
  {
    "workstream": "string",
    "jiraKey":    "string",
    "priority":   "high | medium | low"
  }
]
```

---

## External integrations

| Service | Used by | Auth method | Operations |
|---|---|---|---|
| OpenAI API | Orchestrator, Status, Risk, Comms, Reporting | API key | `POST /v1/chat/completions` |
| Jira Software Cloud | Status, Risk, Comms | Basic auth (email + API token) | GET issues, POST create issue, GET project metadata |
| Airtable | Orchestrator, Risk | Personal Access Token | GET records from Projects and Dependency Mapping tables |
| Google Drive | Comms, Reporting | OAuth2 | Search files, download files, create document from text |
| Gmail | Slack Approval Handler | OAuth2 | Send email |
| Slack Web API | Orchestrator, Slack Approval Handler, Error Notification | OAuth2 Bot Token | `chat.postMessage` |
| Slack Interactivity | Slack Approval Handler | Webhook + HMAC-SHA256 | Receive button payloads |

---

## Security

### Slack request verification

The Slack Approval Handler implements full HMAC-SHA256 signature verification per the [Slack signing secret specification](https://api.slack.com/authentication/verifying-requests-from-slack):

1. Extract `x-slack-signature` and `x-slack-request-timestamp` headers
2. Reject requests with missing headers
3. Reject requests older than 300 seconds (replay attack protection)
4. Compute `HMAC-SHA256(signingSecret, "v0:{timestamp}:{rawBody}")`
5. Compare using `crypto.timingSafeEqual()` (constant-time, prevents timing attacks)

> The verifier node is implemented in `Code in JavaScript1` but is not yet wired into the main flow — it requires a publicly accessible webhook URL. Pending production deployment.

### Credentials

All credentials are stored in n8n's encrypted credential store. No secrets appear in workflow JSON files. Environment variables are used for channel IDs, recipient emails, and signing secrets — never hardcoded in node parameters.

### Jira key validation

The Status Agent validates all Jira keys from the orchestrator LLM output against `/^[A-Z][A-Z0-9]{1,9}$/` before they reach the Jira API. This prevents LLM hallucinations from triggering unintended Jira queries.

---

## Error handling strategy

The system uses a layered error handling approach:

**Layer 1 — Input validation.** Code nodes validate all inputs at workflow entry points before any external API calls are made. Bad data is caught early with descriptive error messages.

**Layer 2 — LLM output validation.** All OpenAI responses are schema-validated before being passed downstream. Required fields, type checks, and enum validation are applied at each parse node.

**Layer 3 — Graceful degradation.** Where possible, agents return safe fallback values rather than throwing. The Status Agent returns AMBER with `_no_data: true` for empty sprints. The Orchestrator falls back to the Airtable project list if GPT-4o returns a bad plan. The Reporting Agent uses "no data — do not invent" prompt instructions when input sections are absent.

**Layer 4 — Continue-on-error.** The Google Drive upload node and the Call Slack node are configured with `onError: continueRegularOutput`. A Drive or Slack failure does not abort the pipeline — the brief is still generated and subsequent steps still execute.

**Layer 5 — Global error notification.** Any unhandled exception in any workflow triggers the Error Notification workflow, which posts a structured Slack alert containing the workflow name, failed node, error type, error message, and a direct link to the n8n execution log.

**Layer 6 — OpenAI retries.** All five OpenAI HTTP Request nodes are configured with `retryOnFail: true`, `maxTries: 3`, and `waitBetweenTries: 5000ms` to handle transient API failures transparently.
