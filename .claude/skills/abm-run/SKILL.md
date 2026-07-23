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
2. Read its `campaign_choices`, `state/permission-ledger.yaml`, `state/pending-approvals.yaml`, and the relevant history/state files.
3. Parse any override arguments (below) into an **ephemeral** run config — never written back to `campaign.yaml`.

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
