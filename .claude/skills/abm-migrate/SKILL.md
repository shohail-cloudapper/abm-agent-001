---
name: abm-migrate
description: Discover previously collected ABM campaign data (local files, CSVs, Drive, Apollo, Monday, logs, reports) and consolidate it into the campaign's ONE permanent master Google Sheet — deduplicated, idempotent, evidence-preserving. Consolidation/cleanup only; never sends, enrolls, publishes, deletes, or creates sequences. Use when the user types /abm-migrate [current|all|status|campaign=ID|force=true].
argument-hint: "[current | all | status | campaign=CAMPAIGN-ID | force=true]"
arguments:
  - mode
user-invocable: true
disable-model-invocation: false
effort: high
---

# /abm-migrate

Migrates historical campaign data into the campaign's canonical master Google Sheet (see `state/master-sheet-registry.yaml`) so the Sheet never starts empty when history exists. **This is a consolidation and cleanup process only** — it reads existing data and writes it into the master Sheet + registries. It never enrolls, sends, publishes, invites, deletes, archives-without-approval, or creates a sequence.

## Modes

| Invocation | Meaning |
|---|---|
| `/abm-migrate current` | Migrate the active campaign. |
| `/abm-migrate all` | Discover and migrate every identifiable campaign. |
| `/abm-migrate status` | Show migration status (read-only) — changes nothing. |
| `/abm-migrate campaign=CAMPAIGN-ID` | Migrate one selected campaign. |
| `/abm-migrate force=true` | Re-run migration after explicit user instruction (still idempotent — see below). |

## Idempotency (hard requirement)

Running the same migration twice must NOT create duplicate rows, tabs, master files, Apollo contacts, or sequences. Enforced by:
- A **source fingerprint** per file/Sheet/dataset (path/id + size + last-modified + row-count hash), stored in `state/migration-registry.yaml` under `source_fingerprints`. A source whose fingerprint is unchanged and already `Completed` is skipped and logged `Already Imported`.
- A **per-record dedup key** (see Contact Deduplication) plus the retained `Original Source` + `Original Row or Record ID` columns, so re-importing the same record updates in place rather than appending.
- `force=true` re-processes sources but still applies both guards — it re-verifies rather than blindly re-appending.

## Capability gate (read this first)

Writing tabs/rows into the Google Sheet requires a **Google Sheets connector**, which is **not currently available** in this environment (only Google Drive file operations). Therefore:
- **Discovery, matching, dedup planning, fingerprinting, and the migration registry/report run now** (read-only over sources).
- **The actual row/tab import into the master Sheet is deferred** until a Sheets connector is authorized — at which point `/abm-migrate` performs the writes. Until then, set each source's status to `In Progress`/`Partial` with a clear "import pending Sheets connector" note; never report rows as imported when they were not.
- The master Sheet file + its locked ID already exist (`state/master-sheet-registry.yaml`), so no duplicate file is ever created regardless.

## Step 1 — Resolve the master Sheet (never duplicate)

Per campaign, before importing anything: check `state/master-sheet-registry.yaml`, then the campaign config, then search Drive by exact Sheet name, then by campaign ID. Reuse the canonical Sheet when found; never create another when a valid one exists. If **multiple** candidate master files exist, list their names, links, last-updated dates, and tab names, and **ask which is canonical — do not merge until the user chooses.** If none exists, create exactly one and record its id immediately.

Ensure the Sheet has all 12 canonical tabs (Tab Index, Campaign Configuration, Daily Scouting, LinkedIn Connection Queue, Posts and Comments, Email Sequence, Work Done, Performance Evaluation, Approval Queue, Errors and Blockers, Source Log, Migration Log). Create only missing tabs.

## Step 2 — Discover historical data

Search the project and connected Google Drive:
```
campaigns/  outputs/  outputs/contact-research/  outputs/content-drafts/
outputs/approval-queue/  outputs/dashboard/  outputs/daily-prospects/
logs/  state/  downloads/  reports/  data/  exports/
```
plus Drive folders, Apollo lists/sequences, Monday (only when `monday_mode` permits reading), existing Sheets, and dashboard records. Inspect campaign config files, campaign IDs/names, file/folder names, batch IDs, and sequence names.

## Step 3 — Match each dataset to a campaign

Matching order (stop at first confident match): 1) exact campaign ID, 2) exact campaign name, 3) saved campaign config, 4) Apollo list/sequence association, 5) Drive campaign folder, 6) batch ID, 7) file path/name, 8) content evidence. **If a campaign can't be identified confidently, do NOT guess** — put it in an **Unmapped Data** section of the report and ask the user.

## Step 4 — Migrate into the correct tabs

Write confirmed records into the master Sheet tabs, preserving original dates and evidence. Add the migration-tracking columns to every imported record: `Original Discovery Date`, `Original Source`, `Original File or Sheet`, `Original Row or Record ID`, `Original Batch ID`, `Migration Batch ID`, `Imported At`, `Import Status`, `Data Confidence`, `Migration Notes`.

- **Daily Scouting** ← all prior contact/prospect records (daily & master prospect CSVs, contact-research outputs, Apollo lists/contacts, Sheets, Excel, approval packages, daily reports, Monday when permitted, LinkedIn queues, prior outreach lists). Preserve the **original discovery date** — do not stamp every row with the migration date.
- **LinkedIn Connection Queue** ← all prior contacts with LinkedIn URLs. Status from **confirmed evidence only**: `Not Prepared` / `Prepared` / `Approved` / `Request Sent` / `Pending` / `Accepted` / `Message Ready` / `Message Sent` / `Hold` / `Reject` / `Suppressed` / `Missing URL` / `Failed` / `Unknown Historical Status`. Evidence of *prepared* ≠ `Request Sent`; missing evidence → `Unknown Historical Status`. Preserve request/acceptance/message dates, source, confirmation evidence, notes. **Never send during migration.**
- **Posts and Comments** ← all prior posts/comments (LinkedIn/X/Reddit/Quora/UKG Community/industry/company + draft/approved/manual/published/failed). Dedup key = **post URL**; no URL → mark `Missing URL`, keep for history, do not treat as a verified live post, do not publish. Preserve post date, discovery date, platform, author, company, profile URL, post URL, summary, draft comment, review decision, publication status, published date, confirmation evidence, source, migration batch. **Never publish during migration.**
- **Email Sequence** ← existing Apollo/Mixmax sequence info (platform, name, id, url, status, created/modified, contact count, sender aliases, steps, subjects, copy, delays, schedule, performance, canonical flag, duplicate status). **Do not create a sequence.** Exactly one valid sequence → mark canonical + save its id in `state/sequence-registry.yaml`. Several → report all, delete/archive nothing, ask which is canonical. None → report `Missing`, direct the user to `/abm-campaign edit`.
- **Work Done** ← prior agent/execution logs, daily reports, task summaries, source-research, approval records, dashboard updates. Columns per the registry spec. Prepared ≠ completed; never infer an external action without evidence.
- **Performance Evaluation** ← historical metrics (Apollo, Mixmax, LinkedIn, Sheets, daily/EOD reports, dashboards, logs, Monday when permitted). Preserve the original metric date; never combine unrelated campaigns; never invent. Label values `Not Available`/`Not Connected`/`Not Tracked`/`Historical Value`/`Live Verified`, with a verification-source field + timestamp.
- **Approval Queue** ← historical approval packages/decisions. A historical `Approve all` does **not** authorize new actions today — historical approvals apply only to their original batch.
- **Errors and Blockers** ← prior errors, preserving dates/evidence.
- **Source Log** ← prior research sources, search queries, URLs, preserving dates/evidence.
- **Migration Log** ← one row per imported source (columns: Migration Batch ID, Migration Date, Campaign ID, Source Type, Source Name, Source Link or Path, Records Found, Records Imported, Records Merged, Records Skipped, Records Unmapped, Errors, Import Status, Notes, Completed At). Statuses: `Not Started`/`In Progress`/`Completed`/`Completed with Warnings`/`Partial`/`Blocked`/`Failed`/`Already Imported`.

## Contact deduplication

Match order: 1) normalized business email, 2) normalized LinkedIn URL, 3) Apollo contact ID, 4) full name + company domain, 5) full name + company + job title. Normalize email casing, LinkedIn URL params, company-domain format, whitespace, name capitalization. **Never merge on similar names alone.** When two records are the same person: keep the most recently verified current title, keep all source references, keep the earliest discovery date, keep the latest outreach status, and **preserve suppression/unsubscribe/bounce/Do-Not-Contact status** — never drop a suppression because another source lacks it. Record which rows were merged.

## Preserve old files (never destroy)

Do not delete or overwrite historical CSVs, reports, logs, Sheets, or approval packages. Create a `Historical Data Archive` folder inside each campaign's Drive folder, but **move files only with explicit user approval.** By default: leave old files where they are, add their links to Migration Log, mark the master Sheet as the new source of truth, and mark old files as `Historical Source`.

## Migration registry

Update `state/migration-registry.yaml` per campaign: `master_sheet_id`, `last_migration_batch`, `migration_status`, `sources_processed`, `source_fingerprints`, `records_imported`, `records_merged`, `records_skipped`, `records_unmapped`, `started_at`, `completed_at`, `errors`. Fingerprint every source; use fingerprint + original record id to prevent duplicate imports.

## Migration report (end of run)

Campaigns discovered · master Sheets created · master Sheets reused · contacts found/imported/merged/skipped · LinkedIn queue records imported · posts found · comments imported · sequences found · canonical sequences selected · work-log entries imported · performance rows imported · approval records imported · unmapped files · blocked sources · errors · **exact master Sheet links** · recommended next action.

## Safety rules (during migration)

Do not: enroll Apollo contacts · send emails · create a new sequence · send LinkedIn requests · publish comments · update external campaign membership · delete duplicates from external platforms · change sender aliases · modify scheduled tasks · mark unverified work as completed. Migration is consolidation only.

## Delegation

Runs as the Campaign Director (main session) coordinating `record-keeper`, `data-cleaner` (normalization/dedup), and read-only connector/Drive lookups. It does not invoke further subagents and performs no outreach.
