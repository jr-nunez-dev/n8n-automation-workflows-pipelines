# 🧹 Data Sheet → Cleanse → Database Loader (n8n)

![Workflow canvas screenshot]([docs/workflow-canvas.png](https://github.com/jr-nunez-dev/n8n-automation-workflows-pipelines/blob/main/Data%20Operations/Data%20Sheet%20-%20Cleanse%20-%20Database%20Loader/Data%20Sheet%20-%20Cleanse%20-%20Database%20Loader.png))
<!-- TODO: add a screenshot of the n8n canvas at docs/workflow-canvas.png -->

---

## 📌 Description

**Data Sheet - Cleanse - Database Loader** is an n8n workflow that runs on a daily schedule, reads employee records from a **Google Sheet**, cleans and normalizes every row in JavaScript, validates each record against a strict rule set, flags duplicate `record_id` values, and then splits the data into two paths: valid rows are **upserted into PostgreSQL in batches of 100**, and rejected rows are converted into a `rejected_rows.csv` file with a human-readable reason for each rejection. In parallel, the workflow builds an import summary (totals, rejection rate, and a per-reason breakdown). It is designed for operations, HR, and data teams who receive messy spreadsheet data and need a reliable, auditable path into a database.

---

## ❗ Problem

Loading spreadsheet data into a database by hand means: open the sheet, scan every row, fix casing and whitespace, check emails and ages, spot duplicate IDs, copy the good rows into a SQL tool, and keep a separate list of the bad rows to send back to whoever owns the data. It is slow, easy to get wrong, and leaves no record of why a row was dropped. This workflow removes the repetitive checking and gives every rejected row a written reason.

---

## ✅ Solution

A schedule trigger starts the run, a Google Sheets node reads the rows, two Code nodes clean and validate each record, a third Code node flags duplicate IDs across the whole dataset, an IF node routes valid rows to a batched PostgreSQL upsert and invalid rows to a CSV export, and a final Code node produces an import summary with counts and rejection reasons.

> 📊 Estimated time savings are in [Business Value](#-business-value-estimates). All figures there are labeled estimates with their assumptions.

---

## 👥 Who Will Use This

- Data engineers who maintain spreadsheet-to-database imports
- HR and people-operations teams loading employee master data
- Operations analysts cleaning recurring sheet exports
- Automation and integration specialists building ETL-style n8n flows
- Data quality and governance teams who need rejection audit trails
- Small businesses without a dedicated ETL tool
- Freelancers and agencies delivering data-cleanup automations to clients
- Recruiters and hiring managers reviewing automation portfolio work

---

## 🗺️ Workflow Overview

```text
[Schedule 00:00]
      │
      ▼
[Google Sheets: employeeData]
      │
      ▼
[Code: clean&Normalize]
      │
      ▼
[Code: validateRecords]
      │
      ▼
[Code: detectDuplicateID]
      │
      ├──────────────────────────────┐
      ▼                              ▼
[IF: isrecordValid?]        [Code: generateImportSummary]
   │ true        │ false
   ▼             ▼
[Split In Batches: 100]   [Set: preprejectedRecords]
   │ loop            │              │
   ▼                 │              ▼
[Postgres: insertRecords]    [Convert to File: rejectedCSV]
   │ (upsert)        │
   └──── back to ────┘
        batch loop
```

---

## 🧩 Components

| Node | Purpose |
|---|---|
| `00:00` | Schedule Trigger that starts the workflow |
| `employeeData` | Reads all rows from the source Google Sheet |
| `clean&Normalize` | Trims text, lowercases email, parses `age` and `active`, adds tracking fields |
| `validateRecords` | Applies per-row validation rules and records rejection reasons |
| `detectDuplicateID` | Flags repeated `record_id` values across all rows |
| `isrecordValid?` | Routes rows: valid → database path, invalid → rejection path |
| `batchValidRecords` | Splits valid rows into batches of 100 |
| `insertRecords` | Upserts each batch into a PostgreSQL table |
| `preprejectedRecords` | Shapes rejected rows and flattens reasons into one string |
| `rejectedCSV` | Converts rejected rows into `rejected_rows.csv` |
| `generateImportSummary` | Computes totals, rejection rate, and reason breakdown |

---

## 🔍 Detailed Process

### 1. ⏰ Trigger: `00:00`

A Schedule Trigger starts the run. The node is named `00:00`, and the workflow timezone is `Asia/Manila`.

**Why:** a nightly run keeps the database in sync without anyone starting it by hand. `TODO: confirm` the interval is set to daily at midnight, since the node's interval block is left at its default.

### 2. 📥 Extract: `employeeData`

Google Sheets node (v4.7) that reads rows from a sheet selected by document ID and sheet ID/name.

```text
documentId: =<PUT_CREDS_HERE>
sheetName:  =<PUT_CREDS_HERE>
```

**Why:** the sheet is the single source of truth, and reading it directly means no manual export step.

### 3. 🧼 Clean: `clean&Normalize`

Code node (`runOnceForEachItem`). Every field is converted to a trimmed string, then typed.

```javascript
const cleanText = (value) => value == null ? '' : String(value).trim();

// age: digits only, otherwise null
if (rawAge !== '' && /^\d+$/.test(rawAge)) age = Number(rawAge);

// active: accepted values
['true', 'yes', '1']  -> true
['false', 'no', '0']  -> false
// anything else      -> null
```

| Field | Cleaning rule |
|---|---|
| `record_id`, `full_name`, `department`, `created_at` | Trimmed |
| `email` | Trimmed and lowercased |
| `age` | Integer if digits only, else `null` |
| `active` | `true` / `false` from the accepted values, else `null` |

The node also adds `is_valid: false` and `rejection_reasons: []` as internal tracking fields.

**Why:** converting bad values to `null` instead of guessing keeps the validation step simple and honest.

### 4. ✅ Validate: `validateRecords`

Code node (`runOnceForEachItem`) that collects every failed rule for a row, not just the first.

| Rule | Rejection reason |
|---|---|
| `record_id` empty | `Missing record_id` |
| `full_name` empty | `Missing full_name` |
| `email` empty | `Missing email` |
| `email` fails `^[^\s@]+@[^\s@]+\.[^\s@]+$` | `Invalid email format` |
| `department` empty | `Missing department` |
| `age` not an integer from 18 to 100 | `Age must be an integer between 18 and 100` |
| `active` not boolean | `Invalid active value` |
| `created_at` not a real `YYYY-MM-DD` date | `Invalid created_at date; expected YYYY-MM-DD` |

`created_at` is checked with a real calendar round trip (`Date.UTC`), so `2025-02-30` is rejected.

**Why:** listing all reasons per row lets the data owner fix a record in one pass.

### 5. 🔁 Deduplicate: `detectDuplicateID`

Code node (run once for all items). It walks the rows in order and keeps a `Set` of seen `record_id` values. The first occurrence passes; later occurrences get `Duplicate record_id within CSV file` appended and `is_valid` set to `false`.

**Why:** this needs the whole dataset at once, so it cannot live in the per-row validation node. Note that the reason text says "CSV file" although the source is a Google Sheet (see [Important Notes](#-important-notes)).

### 6. 🔀 Route: `isrecordValid?`

IF node (v2.3) that checks `$json.is_valid`. The **true** output goes to the database path, and the **false** output goes to the rejection path. The node also feeds `generateImportSummary` in parallel.

**Why:** a single boolean decides the path, so routing logic stays in one place.

### 7. 📦 Batch: `batchValidRecords`

Split In Batches node with `batchSize: 100`. Its **loop** output goes to `insertRecords`; `insertRecords` connects back to it to fetch the next batch. The **done** output is currently unconnected.

**Why:** batching keeps each database call to a controlled size.

### 8. 🗄️ Load: `insertRecords`

PostgreSQL node (v2.7) using the `upsert` operation.

```text
schema: =<PUT_CREDS_HERE>
table:  =<PUT_CREDS_HERE>
columns.mappingMode: defineBelow   (mapping and matching columns are currently empty)
```

**Why:** upsert makes the load safe to re-run, because a record with an existing key is updated instead of duplicated. This only works once the matching column (expected: `record_id`) is configured.

### 9. 🚫 Reject: `preprejectedRecords` → `rejectedCSV`

A Set node keeps the nine fields (`record_id`, `full_name`, `email`, `department`, `age`, `active`, `created_at`, `is_valid`, `rejection_reasons`) and flattens the reasons array with `join('; ')`. A Convert to File node then writes `rejected_rows.csv`.

**Why:** an array does not fit a CSV cell, so joining reasons into one string keeps the file readable.

### 10. 📊 Summarize: `generateImportSummary`

Code node that returns a single summary item:

```json
{
  "status": "COMPLETED | COMPLETED_WITH_REJECTIONS",
  "total_rows": 0,
  "valid_rows": 0,
  "rejected_rows": 0,
  "rejection_rate": 0.0,
  "generated_at": "ISO-8601 timestamp",
  "rejection_breakdown": { "Invalid email format": 0 }
}
```

`rejection_rate` is a percentage rounded to two decimals, and `rejection_breakdown` counts each reason across all rejected rows.

**Why:** a short summary lets someone judge a run's data quality at a glance.

---

## 🧾 Data Schemas

### Input (Google Sheet columns)

| Column | Expected content |
|---|---|
| `record_id` | Unique non-empty ID |
| `full_name` | Non-empty text |
| `email` | Valid email address |
| `department` | Non-empty text |
| `age` | Whole number, 18–100 |
| `active` | `true/yes/1` or `false/no/0` (case-insensitive) |
| `created_at` | `YYYY-MM-DD` |

### Normalized record

```json
{
  "record_id": "string",
  "full_name": "string",
  "email": "lowercase string",
  "department": "string",
  "age": 0,
  "active": true,
  "created_at": "YYYY-MM-DD",
  "is_valid": true,
  "rejection_reasons": []
}
```

This workflow contains no AI agents; all logic is deterministic JavaScript.

---

## 🔌 Integrations

**Google Sheets** is the data source. The `employeeData` node reads rows using an OAuth2 credential.

**PostgreSQL** is the destination. The `insertRecords` node upserts valid rows into the configured schema and table.

---

## 🔐 Credentials and Configuration

| Service | Node | What to configure |
|---|---|---|
| Google Sheets OAuth2 | `employeeData` | Credential (`Sheets - Clean`), document ID, sheet ID/name |
| PostgreSQL | `insertRecords` | Credential (`Postgres - clean`), schema, table, column mapping, matching column |

Credential IDs in the exported JSON belong to the original n8n instance. Re-select or recreate credentials after importing.

---

## ☑️ Configuration Checklist

- [ ] Import the workflow JSON into n8n
- [ ] Select or create the Google Sheets OAuth2 credential
- [ ] Replace `<PUT_CREDS_HERE>` for the spreadsheet document and sheet
- [ ] Confirm the sheet headers match the column names in the schema above
- [ ] Select or create the PostgreSQL credential
- [ ] Replace `<PUT_CREDS_HERE>` for the schema and table
- [ ] Create the target table with matching column types
- [ ] In `insertRecords`, map the columns and set the matching column (`record_id`)
- [ ] Exclude `is_valid` and `rejection_reasons` from the database mapping unless the table has those columns
- [ ] Confirm the schedule interval on the `00:00` node
- [ ] Run once manually with a small test sheet that includes good and bad rows
- [ ] Check the database rows, the rejected CSV, and the summary output
- [ ] Activate the workflow (it is exported as inactive)
- [ ] Increase volume only after testing

---

## ⚠️ Important Notes

- **Duplicate wording:** the duplicate rejection message says "within CSV file", but the source is a Google Sheet. Edit the string in `detectDuplicateID` if that matters to you.
- **Duplicate handling:** the first occurrence of a `record_id` is kept and later ones are rejected. Which row is "first" depends on sheet order.
- **Duplicates are only checked within the sheet.** Records that already exist in the database are handled by the upsert, not by validation.
- **Date format:** `created_at` must already be text in `YYYY-MM-DD` form. Sheet cells formatted as dates may arrive in another format and be rejected. `TODO: confirm` how your sheet returns dates.
- **Age:** values such as `25.5` or `"25 yrs"` become `null` and are rejected.
- **Empty runs:** if the sheet returns no rows, downstream nodes receive no items and the summary is not produced.
- **Rejected file output:** `rejectedCSV` creates the file but nothing sends or stores it yet (see improvements).
- **Summary timing:** `generateImportSummary` runs in parallel with the load, so it reports what was validated, not whether the database write succeeded.
- **Status field:** both non-clean branches of the status logic return `COMPLETED_WITH_REJECTIONS`, so the status only distinguishes "no rejections" from "some rejections".
- **No error handling:** a failed Postgres batch stops the execution. No retry or error workflow is configured.

---

## ▶️ How to Run

1. Complete the [checklist](#️-configuration-checklist).
2. Add a few test rows to the sheet:

   | record_id | full_name | email | department | age | active | created_at |
   |---|---|---|---|---|---|---|
   | 1001 | Ana Reyes | ANA@Example.com | Finance | 29 | yes | 2025-03-14 |
   | 1002 | Ben Cruz | ben@invalid | HR | 17 | maybe | 2025-02-30 |
   | 1001 | Ana Reyes | ana@example.com | Finance | 29 | yes | 2025-03-14 |

3. Click **Execute Workflow** in n8n.
4. Expected result:
   - Row 1001 (first) is upserted into PostgreSQL with the email lowercased.
   - Row 1002 is rejected with: `Invalid email format; Age must be an integer between 18 and 100; Invalid active value; Invalid created_at date; expected YYYY-MM-DD`.
   - The second row 1001 is rejected with `Duplicate record_id within CSV file`.
   - The summary shows 3 total rows, 1 valid, 2 rejected, a 66.67 rejection rate.

---

## 💼 Business Value (Estimates)

These are illustrative estimates, not measurements. Replace the assumptions with your own numbers.

| Item | Assumption / result |
|---|---|
| Rows per run | 1,000 (assumed) |
| Manual review time per row | 30 seconds (assumed) |
| Manual time per run | ≈ 8.3 hours (1,000 × 30 s) |
| Automated time per run | Minutes, depending on sheet size and database speed (`TODO: confirm` by timing a real run) |
| Time saved per month | ≈ 250 hours if run daily (30 × 8.3 h, assumed) |
| Labor value | Time saved × your hourly rate |

The real value is consistency: every row gets the same rules, and every rejection has a recorded reason.

---

## 🚀 Scalability and Upgrades

```text
[Trigger: schedule / webhook / new sheet row]
        │
        ▼
[Read + Clean + Validate]
        │
   ┌────┴─────┐
   ▼          ▼
[Upsert]   [Rejected CSV → Drive / Email / Slack]
   │
   ▼
[Summary → Slack / email / log table]
   │
   ▼
[Error workflow → alert + retry]
```

- Deliver `rejected_rows.csv` by email, Slack, or Google Drive
- Send the import summary to a channel or save it to a log table
- Add an error workflow with alerts and retries
- Check records against existing database rows (for example, email uniqueness)
- Support multiple sheets or tenants with shared validation logic
- Move validation rules into a config table so non-developers can edit them

---

## 📖 Example Use Case

An HR coordinator keeps an employee roster in Google Sheets, edited by several people. Overnight, the workflow reads 400 rows, lowercases emails, rejects 12 rows (bad dates, missing departments, two duplicate IDs), and upserts the other 388 into PostgreSQL in four batches. The coordinator opens the rejected file, corrects the 12 rows with the listed reasons, and the next run loads them.

---

## 🛡️ Data and Operational Considerations

- Employee data is personal data. Confirm your obligations under applicable privacy law (for example, the Philippine Data Privacy Act or GDPR) before processing it.
- Limit access to the sheet, the database, and the rejected CSV, since it contains the same personal fields.
- Rejected rows are not reviewed by a person automatically; someone must act on them.
- The workflow does **not** include: alerting, retries, audit logging of database changes, deletion or opt-out handling, encryption beyond what your services provide, or reconciliation against existing database records.
- Monitor execution history for failures, because none are reported automatically.

---

## 📋 Characteristics

| Capability | Current implementation |
|---|---|
| Source | Google Sheets |
| Destination | PostgreSQL (upsert) |
| Trigger | Schedule (`00:00`), `Asia/Manila` timezone |
| Cleaning | Trim, lowercase email, typed `age` and `active` |
| Validation | 7 field rules plus duplicate `record_id` detection |
| Batch size | 100 rows |
| Rejection handling | Reasons per row, exported as `rejected_rows.csv` |
| Reporting | Summary JSON with counts, rate, and breakdown |
| Error handling | None configured |
| Status | Exported inactive |

---

## ✨ Key Features

- 🧼 Consistent text cleanup and type conversion
- ✅ Multi-rule validation with every failure listed per row
- 🔁 Duplicate ID detection across the whole dataset
- 📦 Batched upserts of 100 rows
- 🚫 Rejected rows exported with readable reasons
- 📊 Import summary with rejection breakdown
- ♻️ Safe to re-run thanks to upsert

---

## 🌳 Node Flow Summary

```text
00:00
└─ employeeData
   └─ clean&Normalize
      └─ validateRecords
         └─ detectDuplicateID
            ├─ isrecordValid?
            │  ├─ true  → batchValidRecords
            │  │          ├─ loop → insertRecords → (back to batchValidRecords)
            │  │          └─ done → (not connected)
            │  └─ false → preprejectedRecords → rejectedCSV
            └─ generateImportSummary
```

---

## 📦 Requirements

- n8n (self-hosted or cloud) with Code, Google Sheets, PostgreSQL, Set, IF, Split In Batches, and Convert to File nodes
- A Google account with access to the source spreadsheet and a configured OAuth2 credential
- A PostgreSQL database and a table matching the record schema
- Network access from n8n to both services

---

## 🏗️ Setup

1. In n8n, choose **Import from File** and select `Data Sheet - Cleanse - Database Loader.json`.
2. Follow the [configuration checklist](#️-configuration-checklist).
3. Test with a small sheet before activating.

---

## 🔧 Recommended Production Improvements

1. **Deliver the rejected file.** Send `rejected_rows.csv` to email, Slack, or Drive so someone can act on it.
2. **Send the summary.** Post `generateImportSummary` output to a channel or log table.
3. **Wire the batch `done` output.** Use it to confirm the load finished before reporting.
4. **Add an error workflow.** Alert on failed executions and enable retries on the Postgres node.
5. **Complete the upsert mapping.** Define columns and the matching key explicitly so reruns are idempotent.
6. **Simplify the status logic.** Two branches currently return the same value.
7. **Handle empty sheets.** Add a check so a run with zero rows still reports.
8. **Make date handling explicit.** Convert sheet dates to `YYYY-MM-DD` before validation.
9. **Fix the duplicate message wording** to say "sheet" instead of "CSV file".

---

## 🎯 Portfolio Value

```text
ETL design + Data validation + JavaScript in n8n Code nodes
+ Batch processing + PostgreSQL upserts + Rejection reporting
```

Relevant for automation, data operations, and integration roles.

---

## 💬 One-Sentence Summary

> An n8n workflow that reads a Google Sheet on a schedule, cleans and validates every record in JavaScript, upserts valid rows into PostgreSQL in batches, and exports rejected rows with clear reasons plus an import summary.

---

## 📄 License

`TODO: confirm` (for example, MIT).
