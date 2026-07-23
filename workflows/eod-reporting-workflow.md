# End-of-Day (EOD) Reporting & Dashboard Workflow

Owned by the ABM Campaign Director (the main session — see `CLAUDE.md`). Runs at each active campaign's `reporting_schedule.eod_report` time (in the campaign's own timezone). Campaign-agnostic: every time, platform, folder, and board ID comes from the campaign's `campaign.yaml`, never from this file.

The EOD report is **separate from and additional to** the morning briefing. Its job is to report what actually happened today — not to launch anything.

## Hard reporting rules (enforced on every EOD run)

These exist so the dashboard never overstates progress:

- Prepared work is **not** completed work — count them in separate fields.
- Enrolled contacts are **not** emails sent.
- Connection requests sent are **not** accepted connections.
- Comments prepared are **not** comments published.
- Opened emails are **not** positive responses.
- Never report platform data (opens, clicks, replies, deliveries) that the platform/connector did not actually confirm — mark it `Not Tracked` / `Not Available` / `Awaiting Sync` instead.
- A contact must never be counted under both Mixmax and Apollo — outreach metrics are per-platform and mutually exclusive.
- Never expose full personal contact details (email, phone) in any artifact that could be shared — see the privacy rule below.
- Label every zero, unavailable, blocked, and pending value explicitly. A blank is not a zero.

## Step 1 — Collect verified activity (Director, read-only)

The Director gathers today's activity for each active campaign from confirmed sources only:

- **Local campaign state:** `state/action-history.json`, `state/contact-history.json`, `state/signal-history.json`, `state/active-campaigns.yaml`, `state/pending-approvals.yaml`, `state/permission-ledger.yaml`.
- **Working outputs:** today's files under `outputs/account-research/<campaign_id>/`, `outputs/contact-research/<campaign_id>/`, `outputs/content-drafts/<campaign_id>/`, `outputs/approval-queue/`, `outputs/manual-actions/`, `logs/`.
- **Monday CRM** (read): campaign records, contact/account records, sync status — via the Monday connector when available.
- **Mixmax** (read): sequence status, recipient counts, and — only if the platform returns them — send/delivery/open/click/reply/bounce/unsubscribe figures.
- **Apollo** (read): sequence status, enrolled counts, per-step figures if returned, and credit balance.
- **LinkedIn** (read): invitation/acceptance/message/comment activity, only when Claude in Chrome can actually confirm it.
- **Google Drive prospect-review sheet** (read): the campaign's review sheet, when configured and reachable.

Each worker agent supplies its EOD rollup in the schema defined at the bottom of its own agent file. The Director does not re-do the workers' research at EOD — it aggregates what they reported plus what the connectors confirm.

## Step 2 — Classify every item

Sort all activity into: **completed** (tool/platform-confirmed), **prepared** (drafted/queued, not executed), **pending** (awaiting owner approval), **blocked** (couldn't proceed — connector/Chrome/platform/data), **failed** (attempted, tool reported failure). These five buckets map directly onto the dashboard's Task Status panel and the reporting rules above.

## Step 3 — Analyze performance (honest, denominator-aware)

Where — and only where — the data exists and shares a consistent denominator: best-performing signal, persona, industry, email step, and most effective LinkedIn activity. When outreach hasn't started or the platform doesn't return the numbers, state that plainly as a data limitation rather than inventing a leader.

## Step 4 — Produce the EOD summary

Write `outputs/dashboard/latest-eod-summary.md` in the format in the "EOD summary format" section below. Also archive a dated copy at `outputs/daily-reports/<YYYY-MM-DD>/eod-summary.md`.

## Step 5 — Update the persistent dashboard

- Update `outputs/dashboard/dashboard-data.json` in place (current snapshot).
- Append today's snapshot to `outputs/dashboard/daily-history.json` (preserve all prior days — never overwrite history).
- Regenerate `outputs/dashboard/dashboard.html` (local fallback) from the current data.
- Update/republish the Artifact per `workflows/dashboard-artifact.md` behavior (attempt Artifact first; fall back to local HTML and say so — never claim an Artifact exists when only local HTML was written).
- The dashboard is a **view**, never the master database. Monday CRM and the Drive prospect-review sheet remain the operational sources.

## Step 6 — Save supporting records to Google Drive (when authorized & connected)

Save the EOD summary and a copy of `dashboard-data.json` to the campaign's Drive folder (`campaign.yaml` → `google_drive.campaign_folder_id`), under the reporting subfolders: `Daily Reports/`, `Dashboard Data/`, `Performance Reports/`. If a required subfolder doesn't exist, create it only within a live Drive authorization. If Drive write isn't authorized, or the connector fails, do **not** claim it saved — write a manual-action file (`workflows/manual-prospecting-workflow.md`) and mark Drive status `Awaiting Sync` on the dashboard.

## Step 7 — Identify tomorrow's priorities

Priority accounts, contacts requiring review, follow-ups due, the pending platform decision, pending approvals, manual actions, and recommended next tasks — each tagged with the responsible agent. These populate the dashboard's "Tomorrow's Plan" panel and the summary's "Tomorrow" section.

## EOD summary format

```
# EOD Summary — <campaign_name> — <YYYY-MM-DD>

## Executive Summary
- What the team accomplished
- Most important result
- Most important problem
- Current campaign status

## Agent Activity  (for each of the 5 agents: assigned / completed / pending / blocked)

## Accounts & Contacts
- Accounts researched · People evaluated · Contacts saved to Drive · Awaiting review
- Approved · Synced to Monday · Duplicates prevented · Existing sales-owned

## Outreach
- Platform selected · Contacts enrolled · Emails scheduled · Emails sent · Replies · Positive responses
- Sender alias: preferred campaign alias · available aliases · selected execution alias · alias selected by · alias verification status · alias verification time · failures caused by alias/mailbox issues (never expose passwords, tokens, or sensitive mailbox config)
- LinkedIn actions · Follow-ups required

## Performance
- Best signal / persona / industry / email step · Most effective LinkedIn activity · Data limitations

## Problems
- Connector failures · Chrome failures · Missing data · Failed tasks · Manual actions

## Tomorrow
- Priority tasks · Contacts requiring review · Follow-ups · Pending approvals · Recommended email platform consideration
```

## Scheduling

Wiring the 11:00 and 19:00 checkpoints to fire automatically uses the connected Scheduled Tasks capability. This file documents *what* the EOD run does; the owner turns on the schedule deliberately once the campaign is live and the actions-required list in the dashboard is clear.
