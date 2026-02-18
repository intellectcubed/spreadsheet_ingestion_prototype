# Project Specifications: Vendor CSV Ingestion Pipeline

## Overview

The system ingests recurring CSV feeds from ~50 known insurance vendors. Each vendor provides customer/policy data in a similar but non-uniform format:

* Column names vary (`acctnum`, `account number`, `acct_id`)
* Column order varies
* Certain fields may be combined (full name) or split (first/middle/last)

The pipeline must:

* Perform initial ingestion of historical files (thousands of rows each)
* Continuously ingest recurring updates as new files arrive
* Append-only ingestion (no upserts)
* Maintain an audit trail for all human corrections
* Support adding new vendor formats over time without custom code per vendor

The target is initially SQLite on a single machine, with eventual migration to SQL Server.

---

## Key Constraints

### Scale

* Files contain thousands of rows
* Manual corrections expected: hundreds overall

### Vendor Model

* Vendors are known, but new vendors may appear occasionally
* Schema drift is expected over time

### Identity & Deduplication

A customer/policy may be identified by combinations of:

* Account ID
* Policy ID
* Name + DOB
* Other vendor identifiers

Ingestion is append-only, so identity resolution is used only for analysis/dedup downstream, not update logic.

### Compliance / LLM Usage

LLM use is restricted. PII may be sent only in highly controlled single-field isolation:

* Account ID alone is acceptable
* Name alone may be acceptable
* Name + account ID together is **not** acceptable

LLM usage is permitted in exactly two controlled scenarios:

1. **Header mapping (onboarding):** When a new vendor is added, LLM may be prompted with *only the CSV headers* (no data rows, no PII) to suggest a mapping file skeleton. A human must approve the result before ingestion begins.
2. **Field-level parsing fallback (ingestion):** When a deterministic parser returns `"ambiguous"` for a single field, LLM may be invoked with *only that single field value*.

Rules:

* Only a **single isolated field value** may be sent to LLM
* No multi-field or row-level PII may be included
* All LLM-interpreted outputs must be flagged for human review
* **Per-file LLM circuit breaker:** If LLM fallback is invoked more than a configurable threshold (`MAX_LLM_CALLS_PER_FILE` or `MAX_LLM_PERCENT_PER_FILE`), LLM fallback is **disabled for the remainder of the file**. Ingestion continues deterministically — additional ambiguous fields are logged to `ingestion_errors` without LLM. An operator alert is generated.

LLM is an **exception handler**, not a primary ingestion mechanism.

---

## Architecture

### Step 1: Canonical Wide Table Schema

A single wide customer-policy table stores normalized data across all vendors:

* Customer identifiers
* Policy identifiers
* Name components
* Address/contact fields
* Metadata about ingestion source

Each row includes provenance:

* `source_file`
* `source_vendor`
* `source_row_number`
* `ingestion_timestamp`

Append-only design guarantees historical traceability.

---

### Step 2: Mapping-Driven Ingestion

Instead of writing 50 custom parsers, each vendor has a mapping file defining:

* Vendor column name → canonical field(s)
* Field-level processors to apply

Example mapping file:

| Vendor Column Name | Canonical Field                                                   | Processor        |
| ------------------ | ----------------------------------------------------------------- | ---------------- |
| acctnum            | account_id                                                        |                  |
| cust_name          | full_name_raw,first_name,middle_name,last_name                    | parse_name       |
| addr               | address_raw,address_street,address_city,address_state,address_zip | parse_address    |
| phone              | phone                                                             | normalize_phone  |
| email              | email                                                             | validate_email   |
| dob                | date_of_birth                                                     | parse_date       |

Key rules:

* **Single-output mapping:** The Canonical Field column contains one field name (e.g., `account_id`). The vendor column value maps directly.
* **Multi-output mapping:** The Canonical Field column contains a comma-separated list (e.g., `full_name_raw,first_name,middle_name,last_name`). The first field receives the raw value; the processor expands into the remaining fields. The processor's output keys must match the canonical field names listed.

Mappings are stored as vendor-editable CSV files at `/mappings/vendor_name.csv`. New vendors are added by creating a new mapping file — not code.

---

### Step 2a: Vendor Onboarding Workflow

Mapping files have a lifecycle: they must be generated, reviewed, and approved before ingestion can occur. This is tracked in the `vendor_mappings` table.

#### Mapping Statuses

* `draft` — Generated (by LLM or manually), not yet reviewed. Ingestion is **blocked** for this vendor.
* `approved` — Reviewed and approved by an operator. Ingestion proceeds normally.

#### Day-1 Bulk Onboarding

For initial setup with many vendors, a CLI command (`onboard`) scans all vendor folders under `/incoming/`:

1. For each vendor folder, read headers from one sample CSV file
2. Call LLM with **headers only** (no data rows, no PII) to generate a draft mapping CSV
3. Save as draft in `/mappings/vendor_name.csv` and record in `vendor_mappings` with `status = "draft"`
4. Alert operator that mappings are ready for review

The operator reviews and edits each draft mapping via the web UI, then approves it. Only after approval will the watcher ingest files for that vendor.

#### Ongoing New Vendor Detection

When the file watcher encounters a vendor folder with no mapping:

1. Auto-generate a draft mapping from the first file's headers (single LLM call, headers only)
2. Save as draft, alert operator
3. Skip ingestion for this vendor until mapping is approved
4. Files remain in `/incoming/vendor/` — they are **not** moved to `/failed/`

This makes new vendor onboarding seamless: no manual trigger needed, but human approval is always required before data flows.

#### Re-onboarding / Schema Drift

If a vendor's CSV headers change significantly, the operator can trigger re-onboarding from the UI. This generates a new draft mapping for review. The previous approved mapping is preserved in the audit trail.

---

### Step 3: LLM Fallback for Parsing Exceptions

When a deterministic handler returns `"ambiguous"`:

* The system may optionally invoke an LLM fallback for that single field value
* This behavior is controlled by a runtime toggle:

```python
ENABLE_LLM_FALLBACK = True | False
```

Example:

* 98% of names parsed successfully via `nameparser`
* Remaining 2% escalated to LLM with only the raw name string

LLM returns structured canonical fields:

```json
{
  "first_name": "John",
  "middle_name": "A",
  "last_name": "Smith",
  "notes": "Detected inverted ordering"
}
```

LLM is **never** invoked on full rows or combined identifiers. LLM is **not** invoked when `status == "invalid"` — invalid fields are logged directly to `ingestion_errors` without LLM.

---

### Step 3a: Human Review of LLM Output

All values interpreted via LLM must be surfaced for human review.

Requirements:

* Any field parsed via LLM is flagged in `ingestion_errors`:
  * `parse_method = "llm"`
  * Review requirement is derived: any record where `parse_method = "llm"` requires human review

* The web UI must provide a queue showing:
  * Raw value
  * LLM suggestion
  * Editable correction
  * Approval workflow

This ensures AI-assisted parsing remains auditable and correctable.

---

### Step 4: Human-in-the-Loop Correction with Audit Trail

Corrections are expected at moderate scale (hundreds). The system must:

* Log ingestion errors to `ingestion_errors`
* Present them in a web UI
* Allow manual correction
* Preserve an audit trail of all changes in `correction_audit_log`

Audit records must capture:

* Original value
* Corrected value
* User
* Timestamp
* Reason/comment

---

### Step 5: Continuous Ingestion Pipeline

Vendor identification is **folder-based**. Each vendor has a dedicated drop folder:

* `/incoming/vendor_a/`
* `/incoming/vendor_b/`

The folder name determines which mapping file to load (e.g., `/mappings/vendor_a.csv`).

A file watcher monitors `/incoming/*/` for new files. On detection:

1. Determine vendor from folder name
2. Check `vendor_mappings` status:
   * **No mapping exists** → auto-generate draft from headers (LLM, headers only), alert operator, skip file
   * **Draft** → skip file, alert operator that approval is pending
   * **Approved** → proceed to ingestion
3. Load corresponding mapping CSV
4. Ingest rows (partial loads are acceptable — see below)
5. Move file to `/processed/vendor/` or `/failed/vendor/`

Files for unapproved vendors remain in `/incoming/` and are not moved to `/failed/`.

**Partial ingestion:** File ingestion is not all-or-nothing. Every row is inserted into `customer_policy_records` (with nulls for fields that failed parsing). Failed fields are additionally logged to `ingestion_errors` — a row with multiple bad fields produces multiple error records sharing the same `source_row`. All rows must be accounted for in `customer_policy_records`.

Rollback is only permitted if the file is structurally invalid (unreadable CSV, missing headers, etc.).

---

### Step 6: Future Migration to SQL Server

Design choices must avoid SQLite-only assumptions:

* Use SQLAlchemy ORM as the DB abstraction layer
* Schema must be compatible with SQL Server types
* Avoid SQLite-specific SQL

---

## Tech Stack

* Python 3.11+
* pandas (CSV parsing)
* watchdog (file monitoring)
* SQLite (prototype DB)
* SQLAlchemy (DB abstraction for future SQL Server migration)
* FastAPI (human correction UI)

---

## Database Schema

### Table: customer_policy_records (append-only)

| Column              | Type       |
| ------------------- | ---------- |
| id                  | INTEGER PK |
| account_id          | TEXT       |
| policy_id           | TEXT       |
| first_name          | TEXT       |
| middle_name         | TEXT       |
| last_name           | TEXT       |
| full_name_raw       | TEXT       |
| date_of_birth       | DATE       |
| email               | TEXT       |
| phone               | TEXT       |
| address_raw         | TEXT       |
| address_street      | TEXT       |
| address_city        | TEXT       |
| address_state       | TEXT       |
| address_zip         | TEXT       |
| vendor              | TEXT       |
| source_file         | TEXT       |
| source_row          | INTEGER    |
| ingestion_timestamp | DATETIME   |

---

### Table: ingestion_errors

| Column          | Type                     |
| --------------- | ------------------------ |
| id              | INTEGER PK               |
| vendor          | TEXT                     |
| source_file     | TEXT                     |
| source_row      | INTEGER                  |
| field_name      | TEXT                     |
| raw_value       | TEXT                     |
| suggested_value | TEXT                     |
| parse_status    | TEXT (ambiguous/invalid) |
| parse_method    | TEXT (local/llm)         |
| status          | TEXT (pending/resolved)  |
| created_at      | DATETIME                 |

---

### Table: vendor_mappings

| Column       | Type     | Notes                     |
| ------------ | -------- | ------------------------- |
| vendor       | TEXT PK  | vendor folder name        |
| mapping_file | TEXT     | path to mapping CSV       |
| status       | TEXT     | `draft` / `approved`      |
| generated_at | DATETIME | when draft was created    |
| approved_by  | TEXT     | operator who approved     |
| approved_at  | DATETIME | when mapping was approved |

---

### Table: file_ingestion_runs

| Column           | Type       | Notes                                  |
| ---------------- | ---------- | -------------------------------------- |
| id               | INTEGER PK |                                        |
| vendor           | TEXT       | vendor folder name                     |
| source_file      | TEXT       | full filename                          |
| started_at       | DATETIME   | ingestion start timestamp              |
| completed_at     | DATETIME   | ingestion end timestamp                |
| total_rows       | INTEGER    | number of rows in file                 |
| rows_ingested    | INTEGER    | rows successfully inserted             |
| error_count      | INTEGER    | total ingestion_errors created         |
| llm_calls        | INTEGER    | number of LLM fallback calls           |
| llm_disabled     | BOOLEAN    | true if circuit breaker tripped        |
| status           | TEXT       | success / success_with_errors / failed |
| operator_message | TEXT       | human-readable summary                 |

---

### Table: correction_audit_log

| Column          | Type       |
| --------------- | ---------- |
| id              | INTEGER PK |
| error_id        | INTEGER FK |
| original_value  | TEXT       |
| corrected_value | TEXT       |
| corrected_by    | TEXT       |
| corrected_at    | DATETIME   |
| comment         | TEXT       |

---

## Mapping File Format

Mappings are stored as CSV files at `/mappings/vendor_name.csv` for nontechnical editing.

* **Vendor Column Name** — column name as it appears in files received from the vendor
* **Canonical Field** — the target field(s) in the database schema
* **Processor** — optional processor name to apply to the field value

Example:

| Vendor Column Name | Canonical Field                                                   | Processor       |
| ------------------ | ----------------------------------------------------------------- | --------------- |
| acctnum            | account_id                                                        |                 |
| cust_name          | full_name_raw,first_name,middle_name,last_name                    | parse_name      |
| addr               | address_raw,address_street,address_city,address_state,address_zip | parse_address   |
| phone              | phone                                                             | normalize_phone |
| email              | email                                                             | validate_email  |
| dob                | date_of_birth                                                     | parse_date      |

For multi-output processors, the Canonical Field column is comma-separated. The first field stores the raw value; the processor populates the remaining fields.

---

## Ingestion Engine

`ingest.py` must:

1. Load mapping config
2. Read CSV via pandas
3. Normalize column names
4. Apply processors
5. Insert all rows into `customer_policy_records` (append-only); use nulls for fields that failed parsing
6. Log failed/ambiguous fields to `ingestion_errors`
7. Disable LLM fallback for the remainder of the file if circuit breaker thresholds are exceeded; continue ingestion deterministically

---

## Field Processor Framework

Processors are registered as:

```python
PROCESSORS = {
  "parse_name":       parse_name,
  "normalize_phone":  normalize_phone,
  "parse_address":    parse_address,
  "validate_email":   validate_email,
  "parse_date":       parse_date,
}
```

Each processor returns:

```python
{
  "output": {...},
  "status": "ok" | "invalid" | "ambiguous",
  "notes": str
}
```

The `status` determines the next action:

* `"ok"` → insert parsed output
* `"ambiguous"` → log to `ingestion_errors`; optionally invoke LLM fallback if `ENABLE_LLM_FALLBACK` is true and circuit breaker has not tripped
* `"invalid"` → log to `ingestion_errors`; **do not** invoke LLM

Underlying libraries:

* Names → `nameparser`
* Phones → `phonenumbers`
* Addresses → `usaddress` / `libpostal`
* Dates → `python-dateutil`
* Email → `email-validator`

---

## File Watcher

`watcher.py` must:

* Monitor `/incoming/*/` for new files
* Determine vendor from folder name
* Check vendor mapping status before ingestion:
  * No mapping → auto-generate draft (LLM, headers only), alert operator, skip file
  * Draft → skip file, alert operator
  * Approved → run ingestion
* Move successfully processed files to `/processed/vendor/`
* Move structurally failed files to `/failed/vendor/`
* Leave files for unapproved vendors in `/incoming/` (do not move to `/failed/`)

---

## Human Review Web UI

FastAPI app must provide:

* **Vendor mapping review:** List of draft mappings pending approval. Operator can view, edit, and approve each mapping before ingestion begins.
* **Ingestion error queue:** Unresolved `ingestion_errors` with ability to correct field values.
* **LLM-interpreted values queue:** Separate queue for AI-interpreted fields requiring human review, with ability to approve or correct LLM outputs.
* **Ingestion run dashboard:** Recent `file_ingestion_runs` with status, error counts, circuit breaker alerts, and links to related errors.

Audit log must capture:

* Original raw value
* LLM suggestion (if applicable)
* Final corrected value
* User + timestamp

---

## Implementation Notes

### LLM Circuit Breaker

Per-file thresholds:

```python
MAX_LLM_CALLS_PER_FILE    = 25
MAX_LLM_PERCENT_PER_FILE  = 0.5   # 0.5% of total rows
```

LLM fallback may only be invoked while both thresholds remain below their limits. If either is exceeded:

* LLM fallback is immediately disabled for the remainder of the file
* Ingestion continues deterministically
* All additional ambiguous fields are logged to `ingestion_errors` normally

When the circuit breaker activates, a clear operator alert must be generated containing:

* Vendor name and file name
* LLM call count and which threshold was exceeded
* Action taken (LLM disabled)
* Recommendation to update the handler or mapping

This alert must appear in `file_ingestion_runs.operator_message`, the UI dashboard, and logs.

Example operator message:

```
Processed 8,412 rows.
Logged 53 unresolved fields.
LLM fallback disabled after 25 calls (threshold exceeded).
Recommendation: update parse_name handler for vendor_a.
```

### Ingestion Run Status Semantics

* `success` → file ingested with zero errors
* `success_with_errors` → file ingested; some unresolved fields logged
* `failed` → file could not be processed structurally (bad CSV, unreadable, missing headers)

### Global LLM Safety Limiter (Optional)

Consider adding a global daily cap to prevent one broken vendor from consuming the entire AI budget:

```python
MAX_LLM_CALLS_PER_DAY = 500
```

### SQLAlchemy for SQL Server Migration

Define all models using SQLAlchemy ORM now, even on SQLite. This allows swapping the DB backend later without rewriting ingestion logic. Avoid SQLite-specific SQL.

### Append-Only Design

Append-only simplifies auditing, rollback, and provenance. Later deduplication and identity resolution can be done analytically downstream.

---

## Deliverables

| File                       | Description                                          |
| -------------------------- | ---------------------------------------------------- |
| `models.py`                | SQLAlchemy schema for all tables                     |
| `ingest.py`                | Ingestion engine                                     |
| `processors.py`            | Field processor library                              |
| `watcher.py`               | File watcher                                         |
| `onboard.py`               | Bulk vendor onboarding CLI                           |
| `app.py`                   | FastAPI human review UI                              |
| `/mappings/vendor_*.csv`   | 10 example vendor mapping files (10–20 rows each, exercising all processors) |
| SQLite schema creation     | Automated schema setup                               |
| `README.md`                | Setup instructions and guide for adding new vendors  |
