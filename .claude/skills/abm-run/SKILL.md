---
name: abm-run
description: Run the daily ABM cycle using the campaign's SAVED settings — no platform questions re-asked. Reads campaign_choices, shows a run summary, then prospects/prepares within those settings and the active authorization. Supports temporary per-run overrides. Use when the user types /abm-run (optionally with overrides).
user-invocable: true
disable-model-invocation: false
argument-hint: "[people | source=.. people=.. linkedin=.. comments=..]"
effort: medium
---

# /abm-run

Settings-driven daily run. Reads every platform/limit choice from the active campaign's `campaign_choices` (set via `/abm-campaign`) and **never re-asks the platform questions**. See "Campaign Choice Rules" in `CLAUDE.md`.

## Startup

1. Resolve the active campaign (`campaigns/*/campaign.yaml` `status: active` + `state/active-campaigns.yaml`).
2. Read its `campaign_choices`, `state/permission-ledger.yaml`, `state/pending-approvals.yaml`, `state/master-sheet-registry.yaml`, and the relevant history/state files.
3. Parse any override arguments (below) into an **ephemeral** run config — never written back to `campaign.yaml`.
4. **Verify the canonical master Google Sheet** (next section) before doing campaign work.

## Master Google Sheet (verify → reuse → never duplicate)

Every campaign has exactly ONE canonical Sheet named `[Campaign Name] — ABM Campaign Master`, tracked in `state/master-sheet-registry.yaml`. It is the primary human **review center** — a view over the day's work, not the master database (Monday CRM + the Drive prospect-review data remain the operational sources). Resolve it at the start of every run, in this exact order:

1. Read the active campaign's `master_sheet_id` from the registry.
2. **Live-check** that exact Sheet (Drive `get_file_metadata` by id).
3. If found → **reuse it.** Never create a second master Sheet.
4. If the saved id can't be found → **search Drive** by the exact campaign name + `— ABM Campaign Master`.
5. Exactly one match → **repair the registry** with that id and reuse it.
6. Several matches → **STOP and ask** the owner which is canonical. Never guess, never create a new one.
7. No match → **create ONE** master Sheet and write its id to the registry immediately.
8. Never create another master Sheet unless the owner explicitly requests a replacement.

Then **verify the 12 canonical tabs** in order — `Tab Index`, Campaign Configuration, Daily Scouting, LinkedIn Connection Queue, Posts and Comments, Email Sequence, Work Done, Performance Evaluation, Approval Queue, Errors and Blockers, Source Log, Migration Log — against the `tab_specification` in the registry. **Create only missing tabs** — never recreate or wipe existing ones, never delete prior days' rows. Keep the `Tab Index` and the registry's per-tab `tab_registry` in sync whenever a tab is added.

## Migration gate (before the first campaign work of the run)

Read `state/migration-registry.yaml` for the active campaign. If its `migration_status` is anything other than `completed` (or `completed_with_warnings`) **and** historical data exists, do NOT silently ignore that history and do NOT create an empty competing master Sheet. Instead show:

> `Historical campaign data has not been fully migrated.`

and recommend:

> `/abm-migrate current`

Let the owner choose to migrate first or proceed. Never erase or replace previous history. When `migration_status` is `completed`, proceed using the canonical master Sheet as the source of truth.

**Capability gate:** building/updating tabs, columns, and formatting needs a Google Sheets connector. If none is available this run, do **not** claim the Sheet was updated — write the day's data to the CSV backups + local dashboard as usual, note "master Sheet tab-write skipped: no Sheets connector" in `Errors and Blockers`/the report, and leave the registry `build_status` truthful. The file + its locked id still prevent duplication regardless.

**Tab naming:** Title Case, clear; never `Sheet1`/`New Sheet`/`Data`/`Other`/`Miscellaneous`. Temporary tabs use `YYYY-MM-DD — [Task Name]` and graduate to a permanent named tab once recurring. The Director may add a genuinely-new recurring tab only after the pre-checks in the registry (`agent_created_tab_rules`) — not a new tab every day.

## Argument parsing (temporary overrides)

- `/abm-run` → use saved settings as-is.
- `/abm-run 100` → a bare number overrides **today's people limit** only (`daily_people_limit = 100` for this run).
- `/abm-run source=apollo people=100 linkedin=25 comments=10` → key=value overrides for this run:
  - `source=` → today's `source_platforms` (comma-separate for multiple, e.g. `source=apollo,ukg_community`)
  - `people=` → `daily_people_limit`
  - `linkedin=` → `linkedin_connection_limit`
  - `comments=` → `linkedin_comment_limit`
  - `email=` → `email_sequence_platform` (`apollo` | `mixmax` | `none`)
  - `monday=` → `monday_mode`
- Overrides apply to the **current run only** and must **not** overwrite campaign defaults — unless the owner explicitly says "Save these as the new campaign defaults," in which case call the `/abm-campaign` save path to persist them.
- If an override names a source not previously enabled, treat it as a one-run choice, but still enforce all safety rules (connector availability, etc.).

## Run summary (always show first)

Before doing any work, display a concise summary and, when overrides are in effect, mark each overridden line:

- Campaign
- Prospect sources (`source_platforms`)
- Email sequence platform (`email_sequence_platform`)
- Monday mode (`monday_mode`)
- Daily people limit (`daily_people_limit`)
- LinkedIn connection limit (`linkedin_connection_limit`)
- LinkedIn comment limit (`linkedin_comment_limit`)
- Storage destinations (`storage_settings`)
- Automation mode (`automation_mode`)
- Preferred sender alias (`preferred_sender_alias`) — shown as a suggestion only; plus the currently available sending aliases when live verification of the email platform is possible.

If the owner wants to change any saved choice, tell them to run **`/abm-campaign edit`** — do not re-ask the questions here.

**Sender alias during `/abm-run`:** prepare contacts, sequence, personalization, and the execution package, and **show** the preferred alias (and available aliases when live-verifiable). Do **not** ask for the final sender decision during early research unless it is needed to configure the sequence, and **never send** during `/abm-run`. In the final execution package include a line: `Sender alias: Awaiting owner selection` — or, if the owner already selected an alias in the same run, `Sender alias: [selected alias]`. The binding alias decision + live verification happens at `/abm-execute`.

## Executing the run

Run the daily cycle in `workflows/daily-abm-workflow.md` using the effective (saved + override) settings, honoring `automation_mode`:

- `review_before_outreach` → scout, verify, store to the configured `storage_settings`, draft sequence content **on the single `email_sequence_platform`**, prepare LinkedIn actions, then present a decision package and **stop before any external write** (same gate as `/abm-daily`).
- `research_and_drafts_only` → research + drafts only; never execute anything external, regardless of authorization.
- `automatic_within_saved_authorization` → may execute **only** the categories covered by an active, unexpired execution authorization in the permission ledger; everything else still requires approval. Door-Keeper is still the gate.

The **LinkedIn comment** task = find up to `linkedin_comment_limit` relevant campaign-topic posts, read each real post, and prepare (or, only under authorization, publish) **one unique relevant comment per post**. Never create standalone posts on the owner's profile.

## Master-Sheet updates during the run (when a Sheets connector is available)

As the run proceeds, write into the canonical master Sheet (never a new file):
- **Daily Scouting** — append every newly researched contact (no duplicates, prior days preserved), with the new-contact defaults: Review Status `Pending Review`, and Apollo/LinkedIn statuses `Prepared Only`.
- **Posts and Comments** — append found posts + draft comments with Review Decision `Pending Review`. **Never publish during `/abm-run`.**
- **Email Sequence** — update from the SAME canonical Apollo/Mixmax sequence; `/abm-run` never creates a new sequence.
- **Work Done** — log each agent function's activity throughout the run (prepared ≠ completed).
- **Performance Evaluation** — update after live results exist; label gaps `Not Available`/`Not Connected`/`Not Tracked`/`Awaiting Sync`; never invent values.
- **Approval Queue / Errors and Blockers / Source Log** — update as decisions arise, failures occur, and sources are used.

## CSV backups (always, even when the Sheet can't be written)

Continue producing the daily + master prospect CSVs, daily + master post/comment CSVs, and the daily performance CSV when configured. CSVs are backups/exports; the master Google Sheet is the human-review interface.

## Campaign Director report (end of run — must include the master Sheet link)

End every `/abm-run` with: campaign name · **clickable master Sheet link** · contacts sourced today · total campaign contacts · contacts added to sequence · emails scheduled or sent · LinkedIn requests sent · remaining LinkedIn connection queue · posts found · comment drafts waiting for review · work completed · performance summary · blockers · decisions required. The owner should be able to open the one master Sheet and understand what the agents did, what contacts were found, what outreach was prepared/sent, what posts need review, what is blocked, and how the campaign is performing — without opening multiple files.

## Safety & validation (hard rules)

- Do **not** ask the platform choices again — read them from config.
- Do **not** silently change platforms.
- Do **not** use a source that isn't in the effective `source_platforms`.
- Do **not** access Monday when `monday_mode: disabled` — even if an authorization would otherwise permit a Monday read. Apply the mode: `read_only_dupe_and_ownership_check` = reads only; `sync_approved_abm_prospects` / `qualified_sales_handoff_only` = writes only for that specific purpose, and still Door-Keeper-gated.
- Do **not** create an Apollo sequence when Mixmax is selected; do **not** create a Mixmax sequence when Apollo is selected; do **not** create both unless explicitly requested.
- Do **not** proceed when a selected connector is unavailable — live-verify first (Apollo: apply the `/abm-status live` freshness rules; if Apollo is the source or the sequence platform and its live check fails, stop and report `Apollo live status: Verification Failed`, don't fall back to cached figures).
- If `email_sequence_platform: none`, prepare no sequence and do no email outreach (sourcing/LinkedIn/storage still run).
- Preserve cross-campaign flexibility — read everything from the campaign's own config.
- **Confirm the current campaign settings before the campaign's first live run.**
- Never present cached connector figures as current for a gated action (prospecting, revealing contacts, creating lists/sequences, enrolling, starting a campaign) — those require a fresh live check.
