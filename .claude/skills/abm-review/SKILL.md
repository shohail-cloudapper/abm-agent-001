---
name: abm-review
description: Read-only daily review report for the active ABM campaign — the Campaign Director collects and reconciles verified results from all agents (Apollo, Drive, CSV, LinkedIn prep, logs, local state), presents the day's review in a fixed simple format, updates the persistent dashboard, and presents approval decisions. Never sources, reveals, enrolls, schedules, sends, invites, publishes, or modifies sequences. Use when the user types /abm-review [date].
argument-hint: "[today | YYYY-MM-DD]"
arguments:
  - date
user-invocable: true
disable-model-invocation: false
effort: medium
---

# /abm-review

The **ABM Campaign Director & Invigilator** (the main session — see `CLAUDE.md`) owns this report. It does **not** delegate reporting to any new agent, and does **not** create a new reporting agent. The Director:

- Collects results from all four worker agents
- Verifies completed actions (tool/platform-confirmed only)
- Reconciles Apollo, Google Drive, CSV, LinkedIn preparation, logs, and local state
- Produces the daily review report (format below)
- Updates the persistent dashboard
- Presents approval decisions and records the owner's answers
- Directs the execution agent **after** approval (via `/abm-execute`, never automatically)

`$date` selects the day: `today` or a date (`2026-07-22`).

## READ-ONLY — hard rule

`/abm-review` performs **no external writes** other than updating the *local* dashboard/report/state view. It must **NOT**: source new contacts · reveal emails · enroll contacts · schedule contacts · send emails · send LinkedIn invitations · publish LinkedIn comments · modify sequences. If any of those is wanted, that is `/abm-execute` (after approval). Comments stay in the Google Sheet until separately approved.

## Data sources (read-only)

Local state (`state/*.json`, `state/*.yaml`), working outputs (`outputs/**`), `logs/**`, and — read-only — Apollo (sequence/enrollment/credits), Google Drive (review sheets/CSVs), Mixmax, and LinkedIn prep files. Never present cached connector figures as current for a gated decision; live-verify Apollo per `/abm-status live` freshness when it affects a decision. When the dashboard and the operational records disagree, **show the discrepancy and use the live operational record as the current value.**

## Report format — present EXACTLY these 10 sections

### 1. Today at a Glance
Show each metric, and clearly separate the buckets **Prepared · Approved · Completed · Failed · Blocked · Waiting for Review**. **Never count prepared work as completed.**
- New qualified contacts sourced · Total contacts reviewed · Contacts approved · Contacts excluded · Contacts added to Apollo · Contacts enrolled · Emails scheduled · Emails sent · LinkedIn connections prepared · LinkedIn connections sent · Relevant LinkedIn posts found · Comment drafts prepared · Comments published · Apollo credits used · Current blockers.

### 2. What Each Agent Did
One short row per agent: **Agent · Assigned task · Completed work · Pending work · Blocker · Next action.** Keep it short and understandable.

### 3. Contact Review
Summarize by: signal-based · technographic · carried-over · high-priority · needs-more-research · hold · rejected · do-not-contact · duplicate · already-contacted · missing-verified-email. Show the **exact Google Sheet and CSV links/paths**. Do **not** dump every contact in chat when the list is large — instead show: total count · top companies · top personas · top industries · exact review-file location.

### 4. Email Sequence Review
Selected email platform · sequence name · sequence status · number of steps · sending schedule · daily email target · contacts ready for enrollment · contacts already enrolled · emails sent · **sender alias status** · missing proof points · sequence risks. Before execution, show `Sender alias: Awaiting owner selection` — never assume/reuse an alias automatically.

### 5. LinkedIn Connection Review
Connection requests prepared · with verified URLs · needing more research · already sent · exact review Sheet/CSV. Do **not** send during `/abm-review`.

### 6. LinkedIn Post and Comment Review
Relevant posts found · verified live post URLs · comment drafts prepared · rows awaiting review · rows Approved to Publish · rows Manual Post · rows on Hold · rejected rows · exact Google Sheet and CSV links. Do **not** publish during `/abm-review`; comments stay in the Sheet until separately approved.

### 7. Storage Locations
Exact clickable links/paths for: Google Drive Master Prospect Review Sheet · Daily prospect CSV (Drive) · Local daily prospect CSV · Local master prospect CSV · LinkedIn Post & Comment Review Sheet · LinkedIn daily comment CSV · Approval package · Execution log · EOD report · Dashboard. If something was promised but not created, mark it **`Missing`** — never claim it exists without a confirmed path/link.

### 8. Problems and Blockers
For each blocker, in plain language: what happened · what it prevents · whether Claude can fix it · whether the owner must fix it · recommended next step.

### 9. Decisions Required
Present a simple decision panel:
- **Email decision:** Approve all eligible email contacts · Approve signal-based only · Approve selected · Hold email execution · Reject the batch.
- **LinkedIn connection decision:** Approve all verified · Approve selected · Hold all · Reject all.
- **LinkedIn comment decision:** Keep all for manual review · Approve selected for Claude publishing · Mark selected Manual Post · Hold all · Reject selected.
- **Sender alias decision:** always ask **"Which sender alias should be used for this execution?"**, show all live available aliases, show the preferred campaign alias first but **never auto-select it**.

### 10. Exact Next Step
End every `/abm-review` report with exactly:
`Reply with your decisions, then run /abm-execute today.`
Do **not** execute automatically.

## Simple decision replies (record the owner's answer; still never auto-execute)

- **`Approve all`** → approve all eligible email contacts + all verified LinkedIn connection requests in the latest package; do NOT approve comment publishing (keep comments stored); **ask for sender alias**; do not execute automatically.
- **`Approve emails`** → approve only eligible email contacts; not connections/comments; ask for sender alias.
- **`Approve connections`** → approve only verified LinkedIn connection requests; not email/comments.
- **`Approve selected`** → ask the owner to provide the selected rows/companies/contacts/tiers.
- **`Approve comments`** → approve only rows marked "Approved to Publish" in the LinkedIn Review Sheet; not email/connections.
- **`Hold`** → do not execute anything.

After any approval, record the decision (approval-queue + permission-ledger note), confirm scope, ask for the sender alias if an email/connection execution is approved, and return the exact `/abm-execute today` command — but do not run it.

## Dashboard

Update the existing persistent dashboard using the same verified information — it is a **view, not the master database** (Drive Sheets, CSVs, Apollo records, local logs remain operational records). The chat report and the dashboard must show the **same numbers**; on disagreement, show the discrepancy and use the live operational record.

## Daily campaign targets (this campaign)

New qualified contacts: 100 · Emails sent: up to 100 · LinkedIn connection requests: up to 25 · LinkedIn posts reviewed: up to 15 · LinkedIn comment drafts: up to 15 · LinkedIn comment publishing: separate approval only · **Sender alias: reconfirm before every execution.**

## Rules
- Read-only; the only writes are to the local dashboard/report/state view.
- The Campaign Director owns this — no new reporting agent.
- Prepared ≠ Completed; enrolled ≠ sent; invite sent ≠ accepted; comment prepared ≠ published.
- Always request the sender alias before any execution; never assume/reuse it.
- Comments cannot be published through `/abm-review`.
