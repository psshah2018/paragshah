# Roadmap

This document describes the planned development path from the current pilot implementation toward a fully agentic program management system.

The baseline is a well-engineered automated reporting pipeline. It runs reliably, validates inputs, handles errors, and posts a weekly brief to Slack. What it is not yet is truly agentic — agents run once per trigger, forget everything between runs, and cannot adapt or take initiative. The roadmap below charts the path from automation to agency across four phases.

---

## Current state

| Capability | Status |
|---|---|
| Weekly cron-triggered pipeline | ✅ Implemented |
| Multi-agent orchestration (Status, Risk, Comms, Reporting) | ✅ Implemented |
| Jira sprint status → RAG assessment per workstream | ✅ Implemented |
| Cross-workstream cascade risk detection | ✅ Implemented |
| Meeting notes → action items → Jira tasks | ✅ Implemented |
| Exec brief generation (GPT-4o) | ✅ Implemented |
| Google Drive brief storage | ✅ Implemented |
| Slack one-click approval gate | ✅ Implemented |
| Email delivery on approval | ✅ Implemented |
| Global error notification | ✅ Implemented |
| Input validation and schema guards | ✅ Implemented |
| DEBUG-gated logging | ✅ Implemented |
| Idempotent Jira ticket creation | ✅ Implemented |
| Graceful fallbacks (no sprint, no projects, no meeting files) | ✅ Implemented |
| Slack signature verification | ⏸ Implemented, pending production wiring |
| Week-over-week status history | ❌ Not implemented |
| PM edit loop | ❌ Not implemented |
| Run deduplication | ❌ Not implemented |

---

## Phase 1 — Close core gaps

**Target:** 2–4 weeks  
**Focus:** Three structural gaps that prevent the system from being fully reliable in production. These are prerequisites for everything that follows.

---

### 1.1 Status history and trend tracking

**Why it matters:** The `velocity_trend` field in every Status Agent output is computed from a single sprint snapshot. The LLM has no historical data to compare against, so "improving" or "declining" is a guess rather than an observed trend. This undermines one of the most valuable signals a PM needs — is this workstream getting better or getting worse?

**What to build:**

Add a `Status History` table to Airtable (see `airtable_schema.md`). After each Status Agent run, write `rag_status`, `days_variance`, `velocity_trend`, and `blocker_count` per workstream, keyed by `workstream + week_start`.

Modify the Status Agent to read the last 4 weeks of history for each workstream and include it in the GPT-4o-mini prompt:

```
Historical status (last 4 weeks):
Week of 2025-04-07: amber, days_variance: 2, velocity: stable
Week of 2025-04-14: amber, days_variance: 3, velocity: declining
Week of 2025-04-21: red,   days_variance: 5, velocity: declining
Week of 2025-04-28: [current run]
```

The LLM can now produce a genuine trend assessment based on observed data.

**Workflows affected:** Status Agent, Orchestrator (to pass history records)  
**New Airtable table:** `Status History`

---

### 1.2 PM edit loop

**Why it matters:** When a PM clicks **Request edit**, the system posts a Slack message and stops. There is no notification to the brief author, no re-trigger of the Reporting Agent, and no way to track that an edit was requested. The approval gate is half-built.

**What to build:**

1. Store the `briefMarkdown`, `overallRag`, and `docUrl` in a `Briefs` Airtable table at the time the brief is generated. Include the Airtable record ID in the Slack Approve button value.
2. On **Approve**: fetch the brief from Airtable, use it to enrich the email body, and mark the record `Status: Approved`.
3. On **Request edit**: send a Slack DM to a configured PM user ID (`SLACK_PM_USER_ID`), re-trigger the Reporting Agent via webhook with an edit instruction payload, and mark the record `Status: Edit Requested`.

**Environment variables to add:** `SLACK_PM_USER_ID`, `AIRTABLE_BRIEFS_TABLE_ID`  
**Workflows affected:** Slack Approval Handler, Reporting Agent  
**New Airtable table:** `Briefs`

---

### 1.3 Run deduplication guard

**Why it matters:** If n8n restarts mid-execution or the Monday cron fires twice, a second pipeline run creates a second Slack message and potentially duplicate Jira tickets for all risks and action items identified that week.

**What to build:**

Add an Airtable check immediately after the `Weekly cron` trigger. Query the `Briefs` table (created in 1.2) for a record with `WeekStart` matching the current Monday. If one exists with `Status` not equal to `Error`, skip the run by routing to a `No Operation` node with a warning log.

```
Weekly cron
  → Check Briefs table (WeekStart = this Monday, Status ≠ Error)
  → If exists: log "Run already completed this week" → No Operation
  → If not exists: proceed to Fetch program context
```

**Workflows affected:** Orchestrator  
**Depends on:** 1.2 (Briefs table)

---

### 1.4 Slack auth wiring (production)

**Why it matters:** The HMAC-SHA256 signature verifier is fully implemented but not connected to the main flow. In the current state, the approval endpoint accepts payloads from any caller.

**What to build:**

Wire the verifier into the main flow as described in `slack_setup.md`. This requires a publicly accessible HTTPS endpoint and a live Slack connection — it cannot be tested locally without a tunnel.

```
Webhook → Code in JavaScript1 (verifier) → Respond to Webhook → Code in JavaScript (parser) → If
```

**Workflows affected:** Slack Approval Handler

---

## Phase 2 — True agentic loops

**Target:** 4–8 weeks  
**Focus:** Move from agents that run once and stop to agents that reason about their own output, act on conditions, and interact with users conversationally.

---

### 2.1 Self-evaluation and retry

**Why it matters:** Currently, agents accept any LLM output that passes schema validation. A brief that is technically valid but factually poor — vague, missing key risks, or poorly structured — is delivered without review.

**What to build:**

Add a reflection step after the Reporting Agent's OpenAI call. A second lightweight GPT-4o-mini call evaluates the brief against a rubric:

- Does it contain a clear RAG status?
- Does it name specific blockers with owners?
- Is it within the 350–450 word target?
- Is every claim supported by the input data?

If the score is below threshold, the agent re-invokes with targeted correction instructions. Maximum 2 retries before falling back to the original output with a quality warning flag. Start with the Reporting Agent where brief quality has the most PM impact.

**Workflows affected:** Reporting Agent

---

### 2.2 Proactive daily RED alerting

**Why it matters:** The system runs once a week. A workstream that flips to RED on Tuesday is invisible until the following Monday brief. A PM needs to know about critical blockers the day they emerge, not six days later.

**What to build:**

Add a lightweight daily cron (07:00 Monday–Friday) that runs a stripped version of the Status Agent scan across all workstreams, checking only for `rag_status === 'red'`. If any workstream has flipped to RED since the last recorded status, post an immediate Slack alert to the exec channel without generating a full brief.

This gives the system genuine initiative — it acts on conditions, not just on schedule.

**New workflow:** Daily Alert Agent  
**Depends on:** 1.1 (Status History, to detect "flipped since last week")

---

### 2.3 Slack thread Q&A

**Why it matters:** When the PM receives the brief, their natural response is to ask follow-up questions: "Why is Auth RED?", "Who owns the infra blocker?", "What decision does the Platform team need by when?" Currently there is no way to ask these questions — the PM must open the full Drive document.

**What to build:**

Add a Slack Events API listener that fires when a message is posted in the exec channel thread containing the brief. Route the message to GPT-4o with the full brief Markdown and the original status/risk/comms source data as context. Return the answer directly in the Slack thread.

The `Briefs` Airtable table (from 1.2) provides persistent brief context across the conversation.

**New workflow:** Brief Q&A Agent  
**Requires:** Slack Events API subscription (`message.channels` scope)  
**Depends on:** 1.2 (Briefs table for brief context retrieval)

---

### 2.4 Decision recommendation engine

**Why it matters:** The exec brief currently describes risks and problems but does not suggest specific decisions. The PM must translate "Auth is RED due to unresolved security sign-off" into "we need a decision: proceed without sign-off, delay the sprint, or escalate to CISO." That translation is exactly the kind of structured reasoning an LLM can do reliably.

**What to build:**

Extend the Risk Agent prompt schema to include a `recommended_decisions` array alongside `risks`:

```json
{
  "recommended_decisions": [
    {
      "description": "Decide whether to proceed with Auth deployment without security sign-off",
      "owner": "Engineering Director",
      "deadline": "2025-04-30",
      "decision_type": "escalate | defer | mitigate | accept",
      "impact_if_delayed": "Auth sprint milestone slips by 5 days"
    }
  ]
}
```

Feed these into the Reporting Agent prompt as a fourth data section. Optionally auto-create Jira tickets labelled `DECISION` for PM triage.

**Workflows affected:** Risk Agent, Reporting Agent

---

## Phase 3 — Autonomous planning

**Target:** 2–4 months  
**Focus:** Move from reporting what happened to actively managing what happens next. Agents plan, forecast, and maintain a continuous model of the program.

---

### 3.1 Sprint planning agent

Build a Sprint Planning Agent that runs before each sprint start. It reads velocity history from the Status History table, fetches the open backlog from Jira, reads team capacity from an Airtable Capacity table, and uses GPT-4o to generate a recommended sprint plan with ticket assignments and a capacity utilisation warning if the team is overloaded. Sends a Slack preview for Scrum Master approval before creating Jira sprints.

**New workflow:** Sprint Planning Agent  
**New Airtable table:** `Team Capacity`  
**Depends on:** 1.1 (velocity history)

---

### 3.2 Dynamic dependency cascade management

Promote the dependency map from a static read-only Airtable lookup to a live managed object. When the Status Agent detects a RED or significantly slipping workstream, the Risk Agent automatically recalculates downstream impact dates using the `lag_days` field, updates `expected_start_date` for affected workstreams in Airtable, and flags them in the next brief — even if their own Jira sprint looks clean.

**Workflows affected:** Risk Agent, Status Agent  
**Depends on:** 1.1 (Status History, to detect slippage magnitude)

---

### 3.3 Stakeholder-personalised communications

Maintain a Stakeholder table in Airtable with each exec's focus areas and communication preferences (e.g. "CTO: technical risks and architecture decisions", "CFO: schedule variance and cost impact", "CPO: product decisions and customer-facing delays"). When the PM approves the brief, generate tailored one-page summaries per stakeholder using GPT-4o and send via Gmail with personalised subjects.

**New workflow:** Stakeholder Comms Agent  
**New Airtable table:** `Stakeholders`  
**Depends on:** 1.2 (Briefs table for approved brief content)

---

### 3.4 Persistent program model

Replace the current architecture — where each weekly run generates a brief from scratch — with a persistent Program Model maintained in Airtable. Each workstream has a live record tracking its current RAG status, milestone dates, active risks, open decisions, and owners. Every weekly run updates this model as a delta. The Reporting Agent reads the delta ("what changed this week") rather than regenerating from raw data. Enables what-if scenario modelling, milestone tracking, and eventual earned-value analysis.

**New Airtable tables:** `Program Model`, `Milestones`, `Decisions`  
**Workflows affected:** All agents

---

## Phase 4 — Multi-agent collaboration

**Target:** 4–6 months  
**Focus:** Agents that communicate with each other, sub-contract tasks, and operate as a genuine multi-agent network with specialised roles and shared context. This is the architecture described in the MIT Agentic AI curriculum.

---

### 4.1 Agent-to-agent task delegation

Allow agents to invoke each other directly mid-run via webhook-triggered sub-workflows. Example: when the Risk Agent detects a cascade risk in workstream B caused by workstream A, it immediately re-invokes the Status Agent for workstream A to get a fresh status before finalising the risk report — without waiting for the next weekly cron.

Task request schema:

```json
{
  "requesting_agent": "risk_agent",
  "task_type":        "status_refresh",
  "context":          { "workstream": "Auth", "jiraKey": "AUTH" },
  "priority":         "high"
}
```

---

### 4.2 Specialist data agents

Add dedicated agents for each external data source the system currently cannot see:

| Agent | Data source | Signal provided |
|---|---|---|
| Finance Agent | Financial system API | Budget burn rate, cost variance, forecast vs actual |
| HR Agent | Capacity management system | Team availability, leave impact, contractor expiry |
| CI/CD Agent | GitHub Actions / CircleCI | Deployment frequency, failure rate, mean time to recovery |

Each specialist agent feeds structured signals into the Risk Agent and Status Agent, widening the system's view of program health beyond Jira sprint data.

---

### 4.3 GPT-4o tool use

Enable select agents to use external tools via GPT-4o function calling, rather than requiring pre-fetched data:

| Agent | Tool | Use |
|---|---|---|
| Risk Agent | `web_search` | Look up industry benchmarks for delivery metrics and compare against program velocity |
| Comms Agent | `calendar_api` | Cross-reference meeting decisions with sprint commitments to detect misalignment |
| Reporting Agent | `previous_briefs` | Write delta-focused updates ("third consecutive week of RED for Auth") rather than re-describing static data |

---

## Version history

| Version | Date | Changes |
|---|---|---|
| v0.1 | 2025-04 | Initial pilot — 7-workflow baseline system. MIT Agentic AI capstone use case. |

---

## Contributing to the roadmap

Issues and pull requests proposing roadmap additions or modifications are welcome. When suggesting a new capability:

- Describe the PM problem it solves, not just the technical feature
- Identify which existing workflows are affected
- Note any new Airtable tables, environment variables, or Slack scopes required
- Indicate dependencies on other roadmap items
