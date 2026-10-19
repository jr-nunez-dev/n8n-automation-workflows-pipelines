# 🌐 [INFRA] Intermittent & Slow Connection Issue Automation

> Screenshot / demo placeholder: `docs/workflow-canvas.png` (n8n canvas) and `docs/jira-and-sheet-result.png` (created ticket + logged row). `TODO: confirm` add before publishing.

---

## 📌 Description

**[INFRA] Intermittent & Slow Connection Issue Automation** is an n8n workflow that watches two Gmail labels for network alert emails (every minute), cleans the email body, suppresses same-day duplicates, routes each alert as *Intermittent* or *Slow Connection*, opens a ticket in Jira Cloud only for fresh (non-reply) emails, then parses the alert into structured fields (date, diagnosis, network device, support group) and appends one row per ticket to a Google Sheet incident log. It is designed for IT NOC and infrastructure support teams that receive monitoring alerts by email and must ticket and track every one.

---

## 🧩 Problem

Without automation, a NOC engineer must open each alert email, judge whether it is intermittent or slow connection, check whether it is a reply to an existing thread or a duplicate of an alert already raised today, copy the subject and body into a new Jira issue, work out which regional support group owns the site, retype the same details into a tracking spreadsheet, and repeat for every alert. The automation removes the copy-paste ticketing, duplicate checking, regional routing, and manual logging.

---

## 💡 Solution

A Gmail Trigger on two labels feeds a shared code node that strips signatures and builds a daily incident key, a Remove Duplicates node drops alerts already seen today, an IF node routes by subject keyword, per-path IF nodes ignore email replies, Set nodes shape the Jira fields, Jira Cloud creates the issue, a Code node parses date, diagnosis, device and support group from the alert text, and Google Sheets appends the enriched record.

**Estimated impact:** see the single canonical table in [Estimated Impact](#-estimated-impact). All figures are labeled estimates.

---

## 👥 Who Will Use This

- IT NOC engineers and shift leads
- Infrastructure and network support teams
- Regional IT technical groups (North, Central, South, NCR, Visayas/Mindanao)
- Service desk coordinators
- Incident and problem managers
- IT operations managers tracking recurring link issues
- Automation developers maintaining the NOC tooling
- Reporting analysts who consume the incident log

---

## 🗺️ Workflow Overview

```text
 Gmail label A            Gmail label B
 (intermittent)           (slowConnection)
        \                      /
         v                    v
        generateDailyIncidentKey
                  |
                  v
         denyDuplicateIncident        (seen today? -> dropped)
                  |
                  v
            intermittent?             (subject contains "intermittent")
             /            \
         true              false
           v                 v
 freshIntermittentEmail?   freshSlowConnectionEmail?
   /          \               /          \
 fresh      reply          fresh        reply
   v          v              v            v
 editData-  disregard-   editData-    disregard-
 ForInter-  Ticket       ForSlow-     Ticket
 mittent    (noOp)       Connection   (noOp)
   v                         v
 createIssue-           createSlow-
 Intermittent           ConnectionIssue
        \                    /
         v                  v
                getData
                   |
                parseData
                   |
              enrichParseData
                   |
                logData  ->  Google Sheet row
```

---

## 🧱 Components

| Node / Module | Purpose |
|---|---|
| `intermittent` (Gmail Trigger) | Polls every minute for new mail under the "intermittent" Gmail label |
| `slowConnection` (Gmail Trigger) | Polls every minute for new mail under the "slow connection" Gmail label |
| `generateDailyIncidentKey` (Code) | Cleans the body and builds `dailyIncidentKey` = issue text + email date |
| `denyDuplicateIncident` (Remove Duplicates) | Drops items whose `dailyIncidentKey` was seen in previous executions |
| `intermittent?` (IF) | Routes by regex match of "intermittent" in the lowercased subject |
| `freshIntermittentEmail?` (IF) | Lets through only emails with no `in-reply-to` header (intermittent path) |
| `freshSlowConnectionEmail?` (IF) | Lets through only emails with no `in-reply-to` header (slow path) |
| `disregardTicket` (NoOp) | Terminal sink for replies on both paths |
| `editDataForIntermittent` (Set) | Maps `subject` and `body` to Jira `subject` / `description` |
| `editDataForSlowConnection` (Set) | Maps `subject` and `body` to `Summary` / `Description` |
| `createIssueIntermittent` (Jira) | Creates the Jira issue for intermittent alerts |
| `createSlowConnectionIssue` (Jira) | Creates the Jira issue for slow connection alerts |
| `getData` (Set) | Re-attaches original subject and raw text to the Jira response |
| `parseData` (Code) | Extracts date/time, diagnosis, location, device, and support group |
| `enrichParseData` (Set) | Builds the final log record, adds ticket key and fixed fields |
| `logData` (Google Sheets) | Appends the record as a new row |

---

## 🔍 Detailed Process

### 1. 📬 Triggers: `intermittent` and `slowConnection`

Two Gmail Triggers poll every minute, each filtered to its own label, with `simple: false` so full headers are available and `maxResults: 1`.

```json
{ "pollTimes": { "item": [{ "mode": "everyMinute" }] },
  "simple": false, "maxResults": 1,
  "filters": { "labelIds": ["<LABEL_ID>"] } }
```

**Why:** `simple: false` is required because later nodes read `headers['in-reply-to']` and `headers.subject`. Label filtering lets mail rules, not the workflow, decide what counts as an alert.

### 2. 🧹 Clean and key: `generateDailyIncidentKey`

Runs once per item. It removes `<email@domain>` brackets and `[image: ...]` placeholders, cuts the body at "Please see attached image", "Regards,", `--`, or "IMPORTANT NOTICE:", then builds the key.

```js
const dateMatch = subject.match(/(\d{8})$/);          // trailing DDMMYYYY-style tag
const baseNetworkIssue = subject.replace(/\s*\|\s*\d{8}\s*$/, '').trim();
const dailyIncidentKey = `${baseNetworkIssue}_${emailDate}`;
```

**Why:** Stripping signatures keeps Jira descriptions clean, and putting the date in the key means the same issue is suppressed within a day but can raise a new ticket on a later day.

### 3. 🚫 Deduplicate: `denyDuplicateIncident`

Remove Duplicates, operation `removeItemsSeenInPreviousExecutions`, value `{{ $json.dailyIncidentKey }}`.

**Why:** Monitoring tools often re-send the same alert; remembering keys across executions stops duplicate tickets without needing a database.

### 4. 🔀 Route: `intermittent?`

```text
{{ $json.subject.toLowerCase() }}  matches regex  "intermittent"
```

True goes to the intermittent path, false to the slow connection path.

**Why:** Both triggers share one pipeline, so routing happens on the subject text. Any subject without the word "intermittent" is treated as slow connection.

### 5. ↩️ Ignore replies: `freshIntermittentEmail?`, `freshSlowConnectionEmail?`, `disregardTicket`

Each IF checks `{{ $json.headers['in-reply-to'] }}` **does not exist**. Fresh emails continue; replies end in `disregardTicket` (NoOp).

**Why:** A reply is a follow-up on an alert already ticketed. Only original emails should open an issue.

### 6. ✏️ Shape Jira fields: `editDataForIntermittent`, `editDataForSlowConnection`

| Path | Fields set |
|---|---|
| Intermittent | `subject` = `$json.subject`, `description` = `$json.body` |
| Slow connection | `Summary` = `$json.subject`, `Description` = `$json.body` |

**Why:** Only subject and the cleaned body should reach Jira; everything else on the Gmail item is noise.

### 7. 🎫 Create tickets: `createIssueIntermittent`, `createSlowConnectionIssue`

Jira Cloud nodes with project and issue type set by ID (placeholders `<PUT_CREDENTIAL_HERE>`), summary from the subject, description from the cleaned body, priority left blank.

**Why:** Two separate nodes keep the paths independent, so project, issue type, or priority can later differ per alert type.

### 8. 🔗 Recover context: `getData`

```text
Subject     = {{ $('intermittent?').item.json.headers.subject }}
Description = {{ $('intermittent?').item.json.text }}
includeOtherFields = true
```

**Why:** The Jira response only contains the issue key and IDs. This node re-attaches the original subject and the **raw** text (not the truncated body), so the alert date is still parseable, and keeps the Jira `key` for logging.

### 9. 🧠 Parse: `parseData`

Deterministic JavaScript, no AI. It:

- normalizes whitespace and strips a leading `Subject:`
- extracts date/time with `(\d{1,2}/\d{1,2}/\d{4}\s+\d{1,2}:\d{2}(?::\d{2})?\s*(?:AM|PM))`
- extracts diagnosis (`Slow Connection` or `Intermittent`) from the subject
- splits the subject on `|`, drops `Status:`, diagnosis, date-tag and `TEST` segments
- treats the segment containing parentheses as the network device, the segments before it as the location
- maps the support group by keyword

| Keyword in subject | Support Group |
|---|---|
| `CNLZ` | IT Technical Central |
| `NLZ` | IT Technical North |
| `SLZ` or `SOUTH LUZON` | IT Technical South |
| `VIZMIN` | IT Technical Visayas/Mindanao |
| `NCR` | IT Technical NCR |
| none of the above | IT Technical Unassigned |

**Why:** Rule order matters: `CNLZ` is tested before `NLZ` so Central is not misread as North. Word boundaries (`\b`) prevent partial matches.

### 10. 🧾 Enrich: `enrichParseData`

Builds the log record (schema in the next section), adding the Jira key from `$('getData').item.json.key` and fixed values for status, domain, and impact.

**Why:** Fixed fields mean every logged row is complete for downstream reporting without the NOC retyping them.

### 11. 📊 Log: `logData`

Google Sheets, operation `append`, auto-map input data, document and sheet placeholders.

**Why:** Auto-map keeps the workflow free of a hard-coded column mapping, so the sheet headers must match the field names exactly.

---

## 📐 Data Schemas

This workflow uses **no AI agent**; all extraction is rule-based. The contracts below replace the AI schema section.

**`parseData` output (per item)**

```json
{
  "summary": "string  (cleaned subject)",
  "dateTime": "string  (e.g. 4/11/2026 10:46 am, or empty)",
  "diagnosis": "string  (Slow Connection | Intermittent | empty)",
  "location": "string",
  "date": "string  (date part of dateTime)",
  "networkDevice": "string",
  "supportGroup": "string"
}
```

**Google Sheet row (`enrichParseData` output)**

| Column | Source |
|---|---|
| Ticket ID | Jira issue `key` |
| Date | `parseData.date` |
| Device Type | fixed: `Network` |
| Network Device | `parseData.networkDevice` |
| Description | `parseData.summary` |
| Status | fixed: `In Progress` |
| Affected IT System | fixed: `NETWORK` |
| Business Impact | fixed: `UNPLANNED & BUSINESS AFFECTING` |
| Diagnosis | `parseData.diagnosis` uppercased |
| Start of Downtime | `parseData.dateTime` |
| Affected IT Domain | fixed: `NETWORK` |
| Support Group | `parseData.supportGroup` |

**Subject convention the parser expects** (illustrative, sanitized):

```text
<REGION> - <Site name> (<DEVICE_ID>) | Status: <state> | Slow Connection | <DDMMYYYY>
```

---

## 🔌 Integrations

**Gmail** is the alert source. Two label-filtered triggers poll the inbox and expose full headers for reply detection.

**Jira Cloud** is the ticketing system. One issue is created per fresh, non-duplicate alert.

**Google Sheets** is the incident log and reporting layer. One row is appended per ticket.

**n8n internal storage** (Remove Duplicates) persists seen incident keys between executions.

---

## 🔑 Credentials & Configuration

| Service | Nodes | Replace / configure |
|---|---|---|
| Gmail OAuth2 | `intermittent`, `slowConnection` | Re-link credential; replace `Label_...` IDs with your own labels |
| Jira Software Cloud API | `createIssueIntermittent`, `createSlowConnectionIssue` | Replace project ID and issue type ID (`<PUT_CREDENTIAL_HERE>`); set priority if wanted |
| Google Sheets OAuth2 | `logData` | Replace document ID and sheet name; create headers matching the schema table |
| n8n settings | Workflow settings | Timezone is `Asia/Manila`; error workflow ID is instance-specific, so re-point or remove it |

Credential IDs in the exported JSON belong to the original instance and will not exist on yours.

---

## ✅ Configuration Checklist

- [ ] Create the two Gmail labels and mail filters that apply them
- [ ] Re-link Gmail, Jira, and Google Sheets credentials
- [ ] Replace both Gmail label IDs
- [ ] Replace Jira project ID and issue type ID on both Jira nodes
- [ ] Decide on a Jira priority (currently blank)
- [ ] Create the Google Sheet with the 12 header columns exactly as listed
- [ ] Replace Sheet document ID and sheet name
- [ ] Set or remove the error workflow
- [ ] Test with a sample intermittent email, a sample slow connection email, a reply, and a repeat of the same alert
- [ ] Activate the workflow; increase `maxResults` or polling limits only after testing

---

## ⚠️ Important Notes

- **One email per poll:** `maxResults: 1` means a burst of alerts inside one minute can be missed or delayed. Test before relying on it.
- **Dedup is persistent:** keys survive across executions. Clearing n8n's stored data re-opens the door to duplicates.
- **Dedup depends on the subject:** if the subject lacks the trailing 8-digit date tag, the key ends with `_` and matches by issue text alone, so a recurring alert on a later day would be suppressed. `TODO: confirm` that every alert carries the tag.
- **Routing is keyword based:** a subject without "intermittent" goes down the slow connection path, even if it arrived under the intermittent label.
- **Parsing is format dependent:** if the date/time pattern or the `|` and parentheses layout changes, `dateTime`, device, or location come back empty and support group falls back to `IT Technical Unassigned`.
- **Dropped field:** `location` is parsed but not written to the sheet.
- **Reply check:** both reply IFs use the `notExists` operator on `in-reply-to`; the right-hand value is not used.
- **Failure handling:** if Jira creation fails, nothing is logged for that alert, since `getData` follows the Jira nodes. `TODO: confirm` what the configured error workflow does.
- **Rate limits:** Gmail polling every minute, plus Jira and Sheets writes, should stay well within normal API quotas at NOC volumes.

---

## ▶️ How to Run

1. Complete the [checklist](#-configuration-checklist) and import the workflow JSON into n8n.
2. Send a test email to the intermittent label with a subject such as `SLZ - Sample Site (SAMPLE_AS01) | Status: Down | Intermittent | 04112026` and a body containing `4/11/2026 10:46 am`.
3. Activate the workflow (or run it manually) and wait one poll cycle.
4. **Expected result:** one Jira issue is created, and one sheet row appears with Diagnosis `INTERMITTENT`, Start of Downtime `4/11/2026 10:46 am`, Support Group `IT Technical South`, Status `In Progress`.
5. Send the same email again: no second ticket. Reply to it: no ticket, the item ends in `disregardTicket`.

---

## 💼 Business Value

All numbers are **estimates** based on the assumptions below. `TODO: confirm` volume and hourly rate with the NOC.

| Assumption | Value |
|---|---|
| Manual effort per alert (read, classify, check duplicate, create Jira, route group, log to sheet) | 5 min (estimate) |
| Alerts per day | 20 (estimate, `TODO: confirm`) |
| Days per month | 30 (estimate, NOC runs daily) |
| Hourly labor rate | `TODO: confirm` |

- Time saved per alert: about 5 minutes
- Time saved per day: 20 x 5 = about 100 minutes
- Time saved per month: about 50 hours
- Labor value per month: 50 hours x hourly rate

---

## 🚀 Scalability & Upgrades

```text
Gmail / monitoring webhook -> dedupe -> classify (rules + optional LLM)
   -> Jira (priority by severity) -> Sheet / DB -> Slack / Teams alert
   -> auto-update and auto-close ticket on recovery email
```

- Replace email polling with a monitoring tool webhook
- Add Slack or Teams notifications to the owning support group
- Set Jira priority and assignee from the support group
- Update or close the Jira issue when a recovery email arrives
- Add more diagnoses (packet loss, link down) through the same router
- Move incident log from Sheets to a database for reporting

---

## 🧪 Example Use Case

A monitoring system emails a "Slow Connection" alert for a south-region site at 8:15 PM. The email lands under the slow connection label, the workflow builds its daily key, finds it new, routes it to the slow path, confirms it is not a reply, and creates a Jira issue with the cleaned body. It then parses `8:15 PM` as the start of downtime, resolves `SLZ` to *IT Technical South*, and logs the ticket. When the same alert is resent an hour later, the key matches and nothing happens.

---

## 🛡️ Data & Operational Considerations

- Alert emails may contain site, device, or contact details. Restrict access to the Jira project and the Sheet.
- The workflow reads only labeled mail, not the whole inbox.
- Do not publish real subjects, device IDs, labels, or credential IDs in a public repository.
- Human review is still needed for severity, priority, and assignment.
- Monitor Jira and Google API quotas, and n8n execution counts.
- **Not included:** ticket updates or closure, notifications, priority/severity logic, retry on failure, escalation, and an opt-out list.

---

## 📋 Characteristics

| Capability | Current implementation |
|---|---|
| Trigger | Gmail polling, every minute, two labels |
| Routing | Subject regex on "intermittent" |
| Duplicate control | Daily incident key + Remove Duplicates (cross-execution) |
| Reply handling | `in-reply-to` header check, replies discarded |
| Ticketing | Jira Cloud, one issue per fresh alert |
| Extraction | Rule-based JavaScript (no AI) |
| Regional routing | Keyword to support group, fallback "Unassigned" |
| Logging | Google Sheets append, 12 columns |
| Timezone | Asia/Manila |
| Status | Exported inactive |

---

## ✨ Key Features

- 📬 Two-label Gmail intake with one shared pipeline
- 🧹 Signature and noise stripping before ticketing
- 🚫 Same-day duplicate suppression that persists across runs
- ↩️ Reply detection so follow-ups never create tickets
- 🎫 Automatic Jira issue creation
- 🧠 Regex parsing of date, diagnosis, device, and location
- 🗺️ Support group routing for five regions with a safe fallback
- 📊 Structured incident log in Google Sheets

---

## 🌳 Node Flow Summary

```text
Triggers
├─ intermittent ─────┐
└─ slowConnection ───┤
                     ▼
generateDailyIncidentKey
└─ denyDuplicateIncident
   └─ intermittent?
      ├─ true  → freshIntermittentEmail?
      │          ├─ fresh → editDataForIntermittent → createIssueIntermittent ─┐
      │          └─ reply → disregardTicket                                    │
      └─ false → freshSlowConnectionEmail?                                     │
                 ├─ fresh → editDataForSlowConnection → createSlowConnectionIssue ─┤
                 └─ reply → disregardTicket                                    │
                                                                               ▼
                                         getData → parseData → enrichParseData → logData
```

---

## 📦 Requirements

- n8n (self-hosted or cloud), version `TODO: confirm` (nodes: Gmail Trigger 1.4, IF 2.3, Set 3.4, Code 2, Remove Duplicates 2, Jira 1, Google Sheets 4.7)
- Gmail account with two labels and mail filters
- Jira Cloud project with permission to create issues
- Google account with a prepared Sheet
- Alert emails that follow the subject convention above

**Setup:** import the JSON via *Workflows → Import from file*, complete the checklist, then activate.

**License:** `TODO: confirm` (for example MIT).

---

## 🔧 Recommended Production Improvements

1. **Raise `maxResults` or switch to a webhook:** avoids missing alerts in bursts.
2. **Add error handling around Jira:** failed creation should be retried and alerted, not silently unlogged.
3. **Set Jira priority and assignee:** use support group and diagnosis to avoid manual triage.
4. **Validate parsed fields:** flag rows with empty date, device, or "Unassigned" group for review.
5. **Log `location`:** it is parsed but currently dropped.
6. **Handle recovery emails:** update the Sheet row and close the Jira issue.
7. **Sanitize the repository:** remove real IDs and sample data before publishing.

---

## 🏆 Portfolio Value

```text
n8n workflow design + Gmail / Jira / Google Sheets integration
+ regex parsing and deterministic data extraction
+ idempotent processing (deduplication, reply filtering)
+ NOC / ITSM process automation
```

Relevant to IT operations, network operations centers, ITSM automation, and internal tooling.

---

## 📝 One-Sentence Summary

> An n8n workflow that turns network alert emails into deduplicated Jira tickets and a structured, region-routed incident log with no manual copy-paste.

---

## 📈 Estimated Impact

| Metric | Estimate | Basis |
|---|---|---|
| Manual time per alert | about 5 min | Assumption, `TODO: confirm` |
| Time saved per month | about 50 hours | 20 alerts/day x 30 days x 5 min |
| Duplicate tickets prevented | `TODO: confirm` | Needs real alert data |
| Support group routing accuracy | `TODO: confirm` | Needs a sample of real tickets |

*Honesty note: every figure above is an estimate from stated assumptions, not a measured result. Replace with real numbers after a trial period.*
