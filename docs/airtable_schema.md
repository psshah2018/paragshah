# Airtable schema

The system uses a single Airtable base (`prg_mgmt_base`) with two tables. This document specifies the exact field names, types, and values required. Field names are case-sensitive — the workflow code references them exactly as written here.

---

## Base

| Property | Value |
|---|---|
| Base name | `prg_mgmt_base` |
| Base ID | Set as `AIRTABLE_BASE_ID` environment variable in n8n |

---

## Table 1: Projects

**Table ID:** `tblLTPVmQ8JriBfui` (your instance will have a different ID — update n8n after creation)  
**Used by:** Orchestrator — `Fetch program context` node  
**Purpose:** Defines the active workstreams the system manages each week. Every record in this table represents one workstream that will be queried in Jira, analysed for status and risk, and included in the exec brief.

### Fields

| Field name | Type | Required | Notes |
|---|---|---|---|
| `Name` | Single line text | Yes | Workstream or project name used in the exec brief and Slack message. Must be non-empty. |
| `JiraKey` | Single line text | Yes | Jira project key — e.g. `AUTH`, `PLATFORM`, `INFRA`. Must match the Jira project key exactly. Case-sensitive. Pattern: `/^[A-Z][A-Z0-9]{1,9}$/` |

> **Important:** The `Build Orchestrator prompt` node filters out any record missing `Name` or `JiraKey` and logs a warning. Records with both fields populated are passed to the orchestrator LLM. Records missing either field are skipped silently — they will not appear in the brief.

### Example records

| Name | JiraKey |
|---|---|
| Platform | PLAT |
| Authentication | AUTH |
| Infrastructure | INFRA |
| Mobile | MOB |

### Notes

- There is no `Active` flag in the current implementation — all records in the table are treated as active. To exclude a workstream temporarily, delete or archive the record in Airtable.
- The table uses n8n's Airtable `search` operation with no filter — it returns all records. If you add a filter in future (e.g. a checkbox field), update the `Fetch program context` node options accordingly.
- The orchestrator LLM may reorder, deprioritise, or group workstreams based on the project list. The final order in the brief reflects the LLM's prioritisation, not the Airtable record order.

---

## Table 2: Dependency Mapping

**Table name in Airtable:** `Dependancy Mapping` *(note: this spelling matches the workflow exactly — do not correct it without updating the Risk Agent node)*  
**Table ID:** `tblShwtvyFdj6HsBW` (your instance will have a different ID — update the Risk Agent node after creation)  
**Used by:** Risk Agent — `Check Project Dependencies` node  
**Purpose:** Defines directed dependencies between workstreams. The Risk Agent reads this table to understand which workstreams depend on which others, enabling cascade risk detection. If workstream A feeds workstream B and A has a critical blocker, the Risk Agent can flag B as at risk even if B's own Jira sprint looks clean.

### Fields

| Field name | Type | Required | Notes |
|---|---|---|---|
| `Upstream workstream` | Single line text | Yes | Name of the upstream (dependency source) workstream. Must match the `Name` field in the `Projects` table exactly. |
| `Downstream Workstream` | Single line text | Yes | Name of the downstream (dependency target) workstream. Must match the `Name` field in the `Projects` table exactly. |
| `Lag Days` | Number | No | Number of days of lag between a delay in the upstream and impact on the downstream. Defaults to `0` if empty. |
| `Type` | Single select | No | Nature of the dependency. Options: `blocks`, `feeds`, `informs`. Defaults to `blocks` if empty. |
| `Notes` | Long text | No | Optional context about the dependency for the LLM. Included in the dependency context passed to GPT-4o. |
| `Active` | Checkbox | No | If set to `false`, the record is excluded from the dependency map. Rows with no `Active` field value are included. |

### Field name precision

> The workflow code references field names with the exact casing shown above. Mismatches will produce `Unknown` values silently. The most common mistake is mismatching the casing of `Upstream workstream` vs `Upstream Workstream`.

The `Assemble Status and Dependencies` node reads these fields as:

```javascript
upstream:        i.json.fields['Upstream workstream']   // lowercase 'w'
downstream:      i.json.fields['Downstream Workstream'] // uppercase 'W'
lag_days:        i.json.fields['Lag Days']
dependency_type: i.json.fields.Type
notes:           i.json.fields.Notes
active:          i.json.fields.Active
```

### Dependency type values

| Value | Meaning |
|---|---|
| `blocks` | Upstream must complete before downstream can proceed. A blocker in upstream directly blocks downstream milestone. |
| `feeds` | Upstream output is consumed by downstream. A delay in upstream slows but does not stop downstream. |
| `informs` | Upstream produces signals or data used by downstream. Impact is indirect — risk of misalignment rather than direct blocking. |

### Example records

| Upstream workstream | Downstream Workstream | Lag Days | Type | Notes |
|---|---|---|---|---|
| Authentication | Platform | 0 | blocks | Auth API must be stable before Platform can complete checkout integration |
| Infrastructure | Authentication | 3 | feeds | Infra Kubernetes upgrade required before Auth can deploy new pods |
| Platform | Mobile | 7 | feeds | Mobile depends on Platform API contract being finalised |
| Infrastructure | Mobile | 5 | informs | Mobile build pipeline runs on infra — degraded CI affects mobile delivery |

### How the dependency map affects risk analysis

The Risk Agent passes the full dependency map to GPT-4o alongside the status records. The LLM is instructed to:

- Flag risks as `critical` when an upstream blocker will impact a downstream milestone within 7 days
- Flag risks as `amber` when a downstream workstream may be impacted within 14 days
- Use `lag_days` to calculate the time window for cascade impact
- Use `type` to calibrate severity — a `blocks` dependency is treated more seriously than an `informs` dependency

If the dependency table is empty, GPT-4o is instructed to analyse each workstream independently and the cascade detection capability is disabled for that run. A warning is logged in the `Assemble Status and Dependencies` node.

---

## Planned tables (Phase 1 roadmap)

These tables are not yet implemented but are required for the Phase 1 roadmap items.

### Status History

Tracks weekly RAG status per workstream. Enables genuine `velocity_trend` analysis by the Status Agent.

| Field name | Type | Notes |
|---|---|---|
| `Workstream` | Single line text | Matches `Projects.Name` |
| `WeekStart` | Date | ISO date of Monday for that week |
| `RagStatus` | Single select | `red`, `amber`, `green` |
| `DaysVariance` | Number | Schedule variance in days |
| `VelocityTrend` | Single select | `stable`, `improving`, `declining` |
| `BlockerCount` | Number | Number of active blockers |
| `NoData` | Checkbox | True if no sprint data was available |

### Briefs

Tracks generated briefs for deduplication, the PM edit loop, and exec email enrichment.

| Field name | Type | Notes |
|---|---|---|
| `WeekStart` | Date | ISO date of Monday for that week |
| `BriefMarkdown` | Long text | Full Markdown content of the brief |
| `OverallRag` | Single select | `red`, `amber`, `green` |
| `DocUrl` | URL | Google Drive link to the saved brief document |
| `GeneratedAt` | Date and time | Timestamp of brief generation |
| `Status` | Single select | `Pending`, `Approved`, `Edit Requested`, `Error` |
