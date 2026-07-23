---
name: abm-campaign
description: Create or edit an ABM campaign's saved settings — prospect sources, email-sequence platform, Monday mode, storage, daily limits, LinkedIn limits, automation mode, schedule, and persona. Interactive; asks one concise question at a time. Use when the user types /abm-campaign or /abm-campaign edit.
user-invocable: true
disable-model-invocation: false
argument-hint: "[edit]"
effort: low
---

# /abm-campaign

Set up or change a campaign's **saved choices**. This skill only reads and writes the campaign's own `campaign.yaml` (the `campaign_choices` block) plus related config — it performs **no** prospecting, no external writes, and no outreach. See "Campaign Choice Rules" in `CLAUDE.md`.

**Core principle:** the prospect **source** and the email-**sequence platform** are separate, independent settings. Never assume they must match.

## Which flow to run

1. Determine the active campaign from `campaigns/*/campaign.yaml` (`status: active`) + `state/active-campaigns.yaml`.
2. **No active campaign / brand-new campaign** → run the **New-campaign flow** (ask the questions below, one at a time).
3. **An active campaign exists AND the user typed plain `/abm-campaign`** → run the **Existing-campaign flow**: first display its saved `campaign_choices`, then ask:
   > "Do you want to keep these campaign settings or change them?"
   Offer exactly these options:
   - Keep current settings
   - Change prospect sources
   - Change email sequence platform
   - Change Monday mode
   - Change daily limits
   - Change LinkedIn limits
   - Change schedule
   - Change persona
   - Change preferred sender alias
   - Review all settings

   Then ask only the question(s) for the chosen area. "Review all settings" walks through every question in order. "Keep current settings" makes no changes.
4. **`/abm-campaign edit`** → skip the summary preamble and go straight to the change menu above.

Ask **one concise question at a time** and wait for the answer before the next. Use `AskUserQuestion` for the multiple-choice questions.

## The questions (exact wording, exact save targets)

### Prospect Sources — save as `source_platforms` (one OR multiple)
> "Where should contacts be sourced from?"
- Apollo
- UKG Community through Claude in Chrome
- ZoomInfo
- LinkedIn or Sales Navigator
- Company websites
- Public web
- Other specified source *(capture the free-text value)*

### Email Sequence Platform — save as `email_sequence_platform` (exactly ONE)
> "Where should the email sequence be created?"
- Apollo
- Mixmax
- No email sequence

Do **not** create sequences in both Apollo and Mixmax unless the owner explicitly requests both (record that as `email_sequence_platform: "both"` only on an explicit request).

### Monday Usage — save as `monday_mode` (one)
> "How should Monday be used for this campaign?"
- Disabled → `disabled`
- Read-only duplicate and sales-ownership check → `read_only_dupe_and_ownership_check`
- Sync approved ABM prospects → `sync_approved_abm_prospects`
- Qualified sales handoff only → `qualified_sales_handoff_only`

### Storage — save as `storage_settings` (one or many; default recommended = all four)
> "Where should the daily prospect data be stored?"
- Google Drive Master Prospect Sheet → `google_drive_master_prospect_sheet` *(default/recommended)*
- Daily Google Drive CSV → `daily_google_drive_csv`
- Local daily CSV → `local_daily_csv`
- Local master CSV → `local_master_csv`

### Daily Limits — ask separately
- "How many people should be sourced daily?" → `daily_people_limit`
- "How many LinkedIn connection requests should be sent daily?" → `linkedin_connection_limit`
- "How many relevant LinkedIn posts should receive comments daily?" → `linkedin_comment_limit`

(When confirming `linkedin_connection_limit`, note that high daily LinkedIn invite volumes carry account-safety risk; don't block the owner's choice, just surface it.)

### Automation Mode — save as `automation_mode` (one)
> "How should the campaign run?"
- Review before outreach → `review_before_outreach`
- Automatic within saved authorization → `automatic_within_saved_authorization`
- Research and drafts only → `research_and_drafts_only`

### Preferred Sender Alias — save as `preferred_sender_alias`
> "Which sender alias should be used as the preferred campaign alias?"
When live verification is possible, show the currently connected mailboxes/aliases in the selected email platform to choose from (e.g. sebastian@cloudapper.ai, shohail@cloudapper.ai, any other connected+authorized alias). Save the answer as `preferred_sender_alias`.
- This is a **preferred suggestion only** — it is **NOT** automatic authorization to use that alias in any future execution.
- Also set `require_sender_confirmation_each_execution: true`.
- The preferred alias may be shown first when aliases are presented, but `/abm-execute` MUST still ask the owner to confirm or change it every run. Never auto-reuse it.

### Schedule / Persona (only when chosen from the change menu)
- Schedule → update `reporting_schedule` (timezone, morning_briefing, eod_report).
- Persona → update `personas` / `buying_committee_roles` as directed.

## Saving

Write the answers into `campaigns/<id>/campaign.yaml` under `campaign_choices`, using the exact field names above. Keep older `daily_defaults` / `daily_limits` in sync with the new limits (raise caps so they never silently block a saved choice). After saving, echo a concise confirmation of the final settings and remind the owner they can run `/abm-run` (settings-driven) or `/abm-campaign edit` to change again.

## Rules
- One question at a time; never dump all questions at once.
- Never assume source == sequence platform. Support every combination in the Campaign Choice Rules table.
- Never trigger prospecting, sequence creation, enrollment, sends, or any external write from this skill — it edits config only.
- Preserve cross-campaign flexibility: write settings into the campaign's own file, never hardcode them into agents or workflows.
- Confirm the final settings before the campaign's first live run.
