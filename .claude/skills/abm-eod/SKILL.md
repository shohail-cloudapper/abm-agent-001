---
name: abm-eod
description: Generate the campaign's End-of-Day report and update the persistent dashboard from verified activity. Reporting-only — never starts new prospecting or outreach. Use when the user types /abm-eod or at the campaign's scheduled EOD time.
user-invocable: true
disable-model-invocation: false
effort: high
---

# /abm-eod

Produces the End-of-Day report and refreshes the persistent dashboard for the active campaign, following `workflows/eod-reporting-workflow.md`. **It never starts new prospecting or outreach and performs no outreach writes** — its only writes are the report/dashboard files and (when authorized) their Google Drive copies + Monday campaign performance fields.

## Read (verified sources only)

Agent activity (from `outputs/*` working files + `logs/`), the Google Drive Prospect Review Sheet, Monday, Apollo, Mixmax, LinkedIn (only when Claude in Chrome confirms it), the scouting logs (`logs/scouting/`), the permission ledger, local state, errors, and open manual tasks. Where a connector is unreachable, label it — never fabricate a metric.

## Update

- Local EOD report: `outputs/dashboard/latest-eod-summary.md` (+ dated archive `outputs/daily-reports/YYYY-MM-DD/eod-summary.md`).
- Google Drive Daily Report (Daily Reports folder) — when Drive write is authorized and reachable.
- Persistent dashboard: `outputs/dashboard/dashboard-data.json` (in place) and append to `outputs/dashboard/daily-history.json` (preserve history), regenerate `outputs/dashboard/dashboard.html`, and re-publish the Artifact (same file path → same URL).
- Monday campaign performance fields on the ABM Campaigns board item — when authorized.

## Report (separate the buckets explicitly)

Accounts researched · people evaluated · contacts waiting for review · contacts approved · contacts synced to Monday · contacts enrolled · emails sent · replies · positive responses · LinkedIn activity · work completed by each of the five agents · pending approvals · failed actions · blockers · tomorrow's priorities.

Every item must be classified as **Completed / Prepared / Awaiting Approval / Failed / Blocked / Manual** — and prepared work is never reported as completed.

## Reporting rules (enforced)

Prepared ≠ completed · enrolled ≠ sent · invitation sent ≠ accepted · comment prepared ≠ published · opened ≠ positive · never report unconfirmed platform data · never sum a contact across Apollo and Mixmax · never expose contact PII in the shareable dashboard · label every zero/unavailable/blocked/pending value.

## Delegation

This is the Campaign Director's own reporting cycle; it aggregates each worker agent's EOD rollup (defined at the bottom of each worker's agent file) rather than re-running their research. It uses `scoreboard-reporter` and `record-keeper`. The dashboard is a view, never the master database — Monday CRM and the Drive prospect-review sheet remain the operational sources.
