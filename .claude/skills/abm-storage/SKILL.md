---
name: abm-storage
description: Show where an ABM campaign's data lives — the canonical master Google Sheet FIRST, then the daily/master CSV backups, reports, dashboard, and logs. Read-only; makes no external writes. Use when the user types /abm-storage [campaign-or-date].
argument-hint: "[campaign-id | date, optional]"
arguments:
  - target
user-invocable: true
disable-model-invocation: false
effort: low
---

# /abm-storage

Read-only map of an ABM campaign's storage. **The canonical master Google Sheet is shown FIRST** — it is the primary human review center. This skill makes no external writes.

(Note: this is the project-level storage skill for this repo. It does not modify, and is separate from, any Cowork `abm-storage` skill.)

## Resolve

1. Determine the target campaign (`$target` if it's a campaign id; else the active campaign from `state/active-campaigns.yaml`). If `$target` is a date, scope the daily CSV/report links to that date.
2. Read `state/master-sheet-registry.yaml` for the campaign's master Sheet, and `campaigns/<id>/campaign.yaml` `external_resources` / `dashboard` for Drive folder + report locations.
3. Optionally do a read-only Drive `get_file_metadata` on the master Sheet id to confirm it still exists and get its last-modified time. Never write.

## Show — master Sheet first

1. **Master Google Sheet (primary):**
   - Master Sheet **name**
   - **Clickable link** (`master_sheet_url`)
   - Sheet **ID**
   - **Required tabs** — which exist vs. which are still to be built (from the registry `tabs_built` / `tabs_planned` + `build_status`)
   - **Additional tabs** present (recommended + any agent-created)
   - **Missing tabs** (required/recommended not yet built)
   - **Last updated** (Sheet last-modified time if the read succeeded, else the registry `last_verified_at`)
   - If the Sheet's tabs aren't built yet, say so plainly and name the blocker (e.g. "Google Sheets connector not available — tab build pending; file + ID exist").
2. **Daily CSV backups:** daily prospect CSV, master prospect CSV, daily post/comment CSV, master post/comment CSV, daily performance CSV (when configured) — with their Drive locations/links.
3. **Reports:** latest EOD summary + daily report locations (local + Google Drive `Daily Reports` / `Performance Reports`).
4. **Dashboard:** local `outputs/dashboard/dashboard.html`, the data files, and the published Artifact URL (from `dashboard-config.yaml`).
5. **Logs:** `logs/` and `logs/scouting/<date>/source-crawl-log.csv`.

## Rules

- Master Sheet first, always — do not bury it under CSVs.
- Every link must be clickable; label anything unconfirmed `Not Available` / `Awaiting Sync`.
- Read-only: never create, rename, move, or write any file or tab. If the master Sheet is missing, point the user to `/abm-run` (which repairs/creates it), don't create it here.
