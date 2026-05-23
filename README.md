# CSC Contract Hours Auto-Submit

Google Apps Script automation for monthly distribution of CSC kidney transplant
surgeon contract hours from a controlled upload point to two paymaster groups
(MSP, VCH), with audit-grade traceability.

See `csc_autosubmit_spec.md` for the build-locked specification.

## Repository layout

```
.
├── csc_autosubmit_spec.md                  build-locked spec
├── csc_analysis_spec.md                    analysis feature spec
├── clinical_article_distribution_spec.md   reference architecture (ArticleRelay)
└── src/
    ├── appsscript.json    manifest (scopes, timezone, webapp access)
    ├── Code.gs            web app entry points (doGet, doPost)
    ├── Config.gs          Script Property + config sheet accessors
    ├── Ledger.gs          audit sheet I/O
    ├── Validate.gs        filename regex + extension whitelist
    ├── Upload.gs          handleUpload() — called from upload.html
    ├── Sendout.gs         runMonthlySendout() — idempotent state machine
    ├── Archive.gs         Archive/YYYY-MM/ helpers
    ├── Sweep.gs           sweepFolder() — 6-hour defense-in-depth sweep
    ├── Notify.gs          notification email subsystem
    ├── Setup.gs           initialSetup() — one-time install
    ├── Utils.gs           helpers
    ├── Analysis.gs        vch-upload analysis pipeline (xlsx → totals → Wide/Long tabs)
    ├── Backfill.gs        bulk historical backfill (editor-only admin)
    └── upload.html        web app UI
```

---

## Configuration Reference

All configuration is stored in **Script Properties** (set via Apps Script editor → Project Settings → Script Properties) and/or **Ledger Sheet tabs**.

### Script Properties

| Property | Required | Description |
|----------|----------|-------------|
| `watched_folder_id` | **Yes** | Google Drive folder ID where uploads land. The folder ID is the string after `/folders/` in the Drive URL. |
| `archive_folder_id` | **Yes** | Google Drive folder ID of the `Archive/` subfolder. Pre-create this inside (or alongside) the watched folder. Monthly subdirs (`Archive/YYYY-MM/`) are auto-created. |
| `ledger_sheet_id` | **Yes** | Google Sheets ID of the audit/ledger spreadsheet. Create an empty sheet first; `initialSetup` creates all tabs. |
| `notification_email` | **Yes** | Email address for all system notifications (upload receipts, sendout summaries, errors). |
| `upload_secret` | **Yes** | Random string embedded in the upload URL for access control. Rotate by editing this property. |
| `timezone` | No | Defaults to `America/Vancouver`. Used for timestamp formatting. |
| `analysis_sheet_id` | No | Google Sheets ID of the analysis output sheet. Enables the vch analysis feature (see below). |
| `backfill_source_folder_id` | No | Drive folder ID for bulk backfill source. Only needed when running `bulkBackfillDryRun()` / `bulkBackfillCommit()`. |

**How to find a Google Drive folder ID:**
```
https://drive.google.com/drive/folders/1abc123xyz...
                                        └─────────┘
                                        This is the folder ID
```

**How to find a Google Sheets ID:**
```
https://docs.google.com/spreadsheets/d/1abc123xyz.../edit
                                       └─────────┘
                                       This is the sheet ID
```

---

## Google Sheets Structure

### Ledger Sheet (audit trail)

Set via Script Property: `ledger_sheet_id`

The ledger sheet contains 7 tabs, all created automatically by `initialSetup`:

| Tab | Purpose | Schema |
|-----|---------|--------|
| `uploads` | Every file uploaded via web UI or sweep | id, drive_file_id, filename, category, name_token, description, extension, size_bytes, uploaded_at, ip_hash, overwrote_prior, sendout_id, status |
| `sendouts` | One row per monthly sendout run | id, run_at, period, msp_count, vch_count, stray_count, msp_email_status, vch_email_status, summary_email_status, archive_status, error_message, completed_at |
| `notifications` | Log of all emails sent by the system | sent_at, subject_prefix, subject, recipient, body_excerpt |
| `surgeons` | Approved surgeon roster (lowercase surnames) | surname (lowercase) |
| `recipients` | Paymaster email addresses by type | type (`msp` or `vch`), email |
| `aliases` | Name variations that map to roster surnames | alias (lowercase), canonical (lowercase) |
| `config` | Runtime tunables (optional) | key, value |

**Populating required tabs:**

1. **`surgeons`** — Add one lowercase surname per row (e.g., `nguan`, `gourlay`). Only files matching these names are accepted.

2. **`recipients`** — Add paymaster addresses:
   | type | email |
   |------|-------|
   | msp | msp-paymaster@example.com |
   | vch | vch-paymaster@example.com |

3. **`aliases`** (optional) — Map common variations to roster names:
   | alias | canonical |
   |-------|-----------|
   | wag | gourlay |
   | cnguan | nguan |

   Run `seedAliases()` from the editor to populate known historical aliases.

### Analysis Sheet (optional)

Set via Script Property: `analysis_sheet_id`

When enabled, each vch xlsx upload extracts per-surgeon hour totals into this sheet. Contains 2 tabs:

| Tab | Purpose | Schema |
|-----|---------|--------|
| `Wide` | One row per period, columns per surgeon | Year, Month, [Surgeon] DI, [Surgeon] C, [Surgeon] DIC, [Surgeon] AHP, ..., Flags |
| `Long` | One row per surgeon-period | Year, Month, Name, DI, C, DIC, AHP, Flags |

**DI computation logic (columns L, M, N from uploaded xlsx):**

The DI (Direct Involvement) hours are extracted from TEMPLATE tab columns:
- **Column L**: Total DI hours (single-column entry)
- **Columns M + N**: Breakdown of DI hours (two-column entry)

| Scenario | DI Value | Flag | Action Required |
|----------|----------|------|-----------------|
| L blank | SUM(M+N) | — | None |
| M+N blank | SUM(L) | — | None |
| SUM(L) = SUM(M+N) ± 0.05 | SUM(L) | — | None |
| **0 < discrepancy ≤ 10 hours** | ***blank*** | `DI discrepancy: L=X, M+N=Y` | **⚠️ MANUAL REVIEW REQUIRED** |
| discrepancy > 10 hours | SUM(L+M+N) | `DI summed all columns (L+M+N=Z)` | Verify user intent |

**⚠️ Important: Small Discrepancy (≤10 hours) Requires Manual Review**

When the difference between SUM(L) and SUM(M)+SUM(N) is between 0.05 and 10 hours:
- The DI and DIC columns are left **blank** in the analysis sheet
- A flag is recorded: `DI discrepancy: L=X, M+N=Y`
- An email notification is sent to `notification_email`
- **You must open the source xlsx and determine the correct value manually**

This safeguard prevents silent data errors when:
- The surgeon made a data entry mistake
- Partial hours were entered inconsistently
- Columns were accidentally double-entered

**Large Discrepancy (>10 hours) — Automatic Sum**

When the discrepancy exceeds 10 hours, the system assumes the surgeon intentionally filled L, M, and N as three separate categories (mutually exclusive entries). The DI is computed as SUM(L) + SUM(M) + SUM(N) and a flag is recorded for audit visibility.

---

## Google Drive Structure

```
[Watched Folder]                    ← watched_folder_id
├── (incoming uploads land here)
└── Archive/                        ← archive_folder_id
    ├── 2025-01/
    │   ├── msp nguan jan2025.xlsx
    │   └── vch gourlay jan2025.xlsx
    ├── 2025-02/
    └── ...
```

---

## Functions Reference

### Core Functions (triggered automatically)

| Function | Trigger | Description |
|----------|---------|-------------|
| `doGet()` / `doPost()` | Web app request | Serve upload UI and handle file submissions |
| `runMonthlySendout()` | Time-driven: 11th of month, 02:00 PT | Email files to paymasters, archive, update ledger |
| `sweepFolder()` | Time-driven: every 6 hours | Defense-in-depth: process any out-of-band uploads |

### Setup & Maintenance (run from editor)

| Function | Purpose | Required Config |
|----------|---------|-----------------|
| `initialSetup()` | One-time install: create tabs, install triggers | All required Script Properties |
| `seedAliases()` | Populate aliases tab with known variations | `ledger_sheet_id` |
| `testAnalysisDiagnostic()` | Debug analysis pipeline failures | `analysis_sheet_id`, Drive API v2 |
| `cleanupTempFiles()` | Remove stray `__csc_tmp_*` conversion artifacts | `watched_folder_id` |

### Backfill Functions (run from editor)

| Function | Purpose | Required Config |
|----------|---------|-----------------|
| `bulkBackfillDryRun()` | Preview what backfill would do (no changes) | `backfill_source_folder_id` |
| `bulkBackfillCommit()` | Actually move files to Archive and populate ledger | `backfill_source_folder_id` |
| `bulkBackfill(folderId, {dryRun})` | Direct call with explicit folder ID | (none — pass ID directly) |

### Preview/Test Functions (safe to run)

| Function | Purpose | Sends To |
|----------|---------|----------|
| `testSummaryEmail()` | Preview sendout summary email | `notification_email` only |
| `testMspEmail()` | Preview MSP paymaster email | `notification_email` only |
| `testVchEmail()` | Preview VCH paymaster email | `notification_email` only |

---

## Deployment

1. Create a new standalone Apps Script project at https://script.google.com.
2. Recreate the file list from `src/` inside the editor:
   - For each `.gs` file: **+ → Script**, then paste contents.
   - For `upload.html`: **+ → HTML**, name it `upload`, paste contents.
   - For `appsscript.json`: **Project Settings → "Show appsscript.json manifest file in editor"** → tick it → edit the manifest tab → paste contents.
3. **Project Settings → Time zone**: set to `America/Vancouver` (mirrors the manifest).
4. **Project Settings → Script Properties** — add all required properties (see Configuration Reference above).
5. **Editor → Services → +** → add **Drive API** (identifier `Drive`, version `v2`) — required for xlsx conversion.
6. Run `initialSetup` from the editor. Approve OAuth scopes when prompted.
7. Populate the `surgeons` and `recipients` tabs in the ledger sheet.
8. (Optional) Run `seedAliases` to populate known name variations.
9. **Deploy → New deployment → Web app**:
   - Description: `CSC Autosubmit v1`
   - Execute as: **Me**
   - Who has access: **Anyone with the link**
   Copy the URL and distribute to surgeons.

See §13 of the spec for the full checklist and §14 for the testing plan.

---

## Operations

### Automatic Triggers

| Trigger | Schedule | What it does |
|---------|----------|--------------|
| Monthly sendout | 11th of month, 02:00 PT | Emails files to paymasters, archives to `YYYY-MM/`, updates ledger |
| Sweep | Every 6 hours | Defense-in-depth: catches out-of-band uploads |

### Notifications

- **Per-upload receipt** — sent to `notification_email` immediately on every successful submission
- **Sendout summary** — sent after each monthly sendout completes
- **Paymaster emails** — BCC'd to `notification_email` for audit trail
- **Analysis flags** — emailed when DI discrepancies require review

### Recovery

If a sendout fails partway, re-run `runMonthlySendout` from the editor. Each step is gated on its own status in the `sendouts` tab, so completed steps are skipped automatically.

### Internal Functions

Do not run any function whose name starts with `_` directly — those are internal helpers and will throw a clear error if invoked without arguments.

---

## Enabling the Analysis Feature

The analysis feature extracts per-surgeon hour totals (DI, C, DIC, AHP) from each vch xlsx upload. See `csc_analysis_spec.md` for the full spec.

**Setup:**

1. Create an empty Google Sheet in any Drive location.
2. Copy its ID from the URL.
3. **Project Settings → Script Properties → Add**: `analysis_sheet_id` = the copied ID.
4. **Editor → Services → +** → enable **Drive API** (identifier `Drive`, version `v2`).
5. Re-run `initialSetup` from the editor — it will create the `Wide` and `Long` tabs.
6. Upload a sample vch xlsx via the web UI to confirm rows appear.

The feature is opt-in: if `analysis_sheet_id` is unset, extraction is a silent no-op.

---

## Bulk Backfill (Historical Import)

To import historical files into the archive with proper ledger entries:

**Setup:**

1. Create a temporary Drive folder containing the files to import.
2. **Project Settings → Script Properties → Add**: `backfill_source_folder_id` = the folder ID.

**Execution:**

1. Run `bulkBackfillDryRun()` from the editor to preview (no changes made).
2. Review the email summary for any skipped/errored files.
3. Run `bulkBackfillCommit()` to actually move files and populate ledger.

Files are routed to `Archive/YYYY-MM/` based on column E (SERVICE START DATE) in each xlsx.
