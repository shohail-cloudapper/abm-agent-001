# Daily ABM Workflow

This is the sequence the main session (acting as ABM Campaign Director, per `CLAUDE.md`) follows for a daily or on-demand run. It's campaign-agnostic — nothing here names a specific product, account, or channel.

## Sequence

1. **Load active campaigns.** Read every `campaigns/*/campaign.yaml` with `status: active`. Skip `_template` and anything `draft`/`paused`/`completed`.
2. **Read campaign state.** For each active campaign: `state/active-campaigns.yaml`, `state/action-history.json`, `state/contact-history.json`, `state/signal-history.json`.
3. **Read the permission ledger.** `state/permission-ledger.yaml` — what's currently authorized, and what's expired.
4. **Review previous actions and pending approvals.** `state/pending-approvals.yaml` and the most recent `outputs/daily-reports/` entry.
5. **Prepare the daily plan.** For each campaign, decide what's needed today: new signal research, contact refresh, content drafting, outreach preparation/execution, follow-up review.
6. **Delegate signal research.** Invoke `account-signal-intelligence` with the campaign's target accounts and objective. Independent accounts may be researched in parallel.
7. **Delegate buying-committee work.** Invoke `account-buying-committee` for accounts that came back from step 6 with a usable signal, or that are due for a contact refresh. This can run in parallel with step 6 for accounts that don't depend on a fresh signal first.
8. **Match signals to contacts.** The Director combines the two workers' output into one per-account, per-contact working set, removing duplicates and anything in cooldown/suppression.
9. **Delegate content drafting.** Invoke `personalization-campaign-content` with the matched signal + contact packages. This must wait for steps 6–8 to produce something to personalize against.
10. **Door-Keeper review.** Before anything proposed by step 9 can become a live action, run `door-keeper` against the campaign's `approval_mode`, daily limits, and the permission ledger.
11. **Delegate outreach preparation/execution.** Invoke `outreach-engagement-followup` with only the drafts and contacts that cleared step 10. It prepares everything; it executes only what's already authorized in `state/permission-ledger.yaml`.
12. **Capture confirmation evidence.** Every executed action must come back with tool/platform confirmation, not an assumption of success.
13. **Update state.** Write results into `state/action-history.json`, `state/contact-history.json`, `state/signal-history.json`, `state/active-campaigns.yaml`, and clear/update `state/pending-approvals.yaml`.
14. **Produce the daily report.** Write `outputs/daily-reports/<YYYY-MM-DD>/report.md` (format below).
15. **Produce one consolidated approval request.** Anything from step 11 that needed authorization but doesn't have it yet becomes a single batch request in `outputs/approval-queue/`, not one request per item.

## Parallelism rule

Independent research tasks (different accounts, different campaigns) may run concurrently. Nothing in steps 9–12 may proceed for a given account/contact until its research, duplicate check, suppression check, and permission check are complete — speed never skips a safety step.

## Daily report format

`outputs/daily-reports/<YYYY-MM-DD>/report.md` must contain:

- **Executive summary** — campaigns reviewed, most important developments, highest-priority accounts, decisions required from the owner.
- **New signals** — from step 6, with evidence links and scores.
- **New / updated contacts** — from step 7, with verification status.
- **Content prepared** — from step 9, with trigger and approval status.
- **Browser actions prepared vs. completed** — from step 11, with confirmation evidence for anything marked complete.
- **Engagement detected** — replies, acceptances, meetings, from `follow-up-buddy`.
- **Follow-ups required** — next actions and owners.
- **Pending batch approval** — pointer to the current `outputs/approval-queue/` entry, if any.
- **Suppressed actions** — anything blocked by suppression/cooldown, with reason.
- **Manual actions** — pointer to anything written to `outputs/manual-actions/` this run.
- **Tool or connector failures** — anything that didn't work, and why.
- **Decisions required from the owner.**
- **Next-run priorities.**

## Daily review report — `/abm-review` (Director-owned, read-only)

The Director presents the day's review via `/abm-review [today|YYYY-MM-DD]` (skill: `.claude/skills/abm-review/SKILL.md`). It is **read-only** (no sourcing, reveal, enroll, schedule, send, invite, publish, or sequence edit — only the local dashboard/report view is written) and uses the fixed 10-section format: 1 Today at a Glance · 2 What Each Agent Did · 3 Contact Review · 4 Email Sequence Review · 5 LinkedIn Connection Review · 6 LinkedIn Post & Comment Review · 7 Storage Locations · 8 Problems & Blockers · 9 Decisions Required · 10 Exact Next Step. Prepared work is always separated from completed work; the sender alias is always requested before execution and never auto-selected; comments are never published from the review. Every review ends with `Reply with your decisions, then run /abm-execute today.` This is the same Director role (no new reporting agent).

## Cadence

This workflow is meant to run once per day per active campaign, and can also be run on demand. Wiring it to an actual daily schedule uses the connected Scheduled Tasks capability (see `docs/tool-and-connector-inventory.md`) — this project doesn't auto-schedule itself; that's a deliberate step the campaign owner takes when a campaign is ready to go live.

## Daily checkpoints (per campaign, when the campaign defines a `reporting_schedule`)

Some campaigns define a `reporting_schedule` in their `campaign.yaml` with a morning-briefing time and an end-of-day time (and a timezone). When present, the single daily run above is framed by two Director-owned checkpoints. Times and timezone are read from the campaign config — never hardcode them here.

**Morning briefing (campaign's `reporting_schedule.morning_briefing`, e.g. 11:00 in the campaign's timezone):**
1. Review yesterday — read the previous day's EOD summary and the campaign's state.
2. Present today's plan — the per-campaign plan from step 5 above.
3. Ask the owner which email platform to use today. Reset the campaign's `daily_email_platform_selection` to `pending_user_selection` at the start of the cycle and surface the choice (values allowed by the campaign, e.g. `mixmax` / `apollo` / `no_email_outreach`). No email enrollment happens until the owner answers.

**During the day:** agents research, review, prepare, and execute only already-authorized tasks (steps 6–12 above), within the campaign's approval mode and the permission ledger.

**End-of-day report (campaign's `reporting_schedule.eod_report`, e.g. 19:00 in the campaign's timezone):** the Director runs the End-of-Day reporting workflow — see `workflows/eod-reporting-workflow.md`. This collects verified activity, analyzes performance, produces the EOD summary, updates the persistent dashboard, and saves supporting records locally and (when authorized and connected) to Google Drive.

The EOD report is separate from and additional to the morning briefing. Neither checkpoint is a license to bypass the approval flow or the permission ledger.
