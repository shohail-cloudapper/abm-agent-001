---
name: abm-status
description: Show the current ABM campaign status — run state, connector health, contact pipeline, approvals, and blockers — without doing any prospecting or external writes. Use when the user types /abm-status or asks "where does the campaign stand right now".
user-invocable: true
disable-model-invocation: false
argument-hint: "[live|cached]"
effort: low
---

# /abm-status

Read-only status snapshot for the active ABM campaign(s). **This skill never sends, publishes, enrolls, or performs any outreach.** It reads local state and, in `live` mode, performs **read-only** connector lookups only. The only writes it ever makes are to *local* campaign state and the *local* dashboard, to record what a live check found (never to an external system).

## Arguments / modes

| Invocation | Meaning |
|---|---|
| `/abm-status live` | Force a live, read-only re-verification of every reachable connector (Apollo, Monday CRM, Google Drive, Mixmax) before reporting. |
| `/abm-status cached` | Report from saved local state only — make **no** connector calls. Every connector value must be stamped with its last-verified time, connector used, data age, and Historical/Current status. |
| `/abm-status` (no arg) | **Live by default** for Apollo, Monday CRM, Google Drive, and Mixmax. Where a live check is not possible, fall back to cached and clearly label it as cached with its last verification time. |

Accept case-insensitive `live`/`cached`. Any other argument → treat as no-arg (live-by-default) and note the unrecognized argument.

## Apollo is the special case — handle it correctly every time

Apollo figures (account, plan, credits, mailbox, sequence capacity, connector availability) go stale and are gating for real actions, so Apollo has stricter handling than the other connectors.

**Connector to call:** the currently connected Apollo MCP connector — server `b4eb3cd7-8008-4759-8812-16897005eddd`. If a second Apollo connector is present (e.g. `plugin:sales:apollo`), it is a duplicate; use only the authorized/paid one and note the duplicate. Never call the obsolete/free-account connector.

### `/abm-status live` (and Apollo in no-arg mode) MUST:

1. **Call the connected Apollo connector** with read-only tools only:
   - `apollo_users_api_profile` (`include_credit_usage: true`, `include_waterfall_capability: true`) — identity, plan-evidence, waterfall flags.
   - `apollo_usage_stats_credit_usage_stats` — authoritative team credit categories + reset cycle.
   - `apollo_email_accounts_index` — connected mailboxes.
   - `apollo_emailer_campaigns_search` — existing sequences + capacity signal.
   - `apollo_emailer_schedules_index` — sending schedule.
2. **Retrieve fresh** account, plan, credit, mailbox, and sequence information from those calls only.
3. **Record the connector used** (the exact server id) in the report and in the files updated.
4. **Record the verification timestamp** (UTC; get it fresh, e.g. `date -u +"%Y-%m-%dT%H:%M:%SZ"`).
5. **Update local campaign state and the dashboard** with the live figures:
   - `campaigns/<id>/campaign.yaml` → `external_resources.apollo`
   - `outputs/dashboard/dashboard-data.json` and `outputs/dashboard/dashboard.html` (Apollo section, platform timestamp, alerts, `_meta` note)
   - the Apollo inspection report under `outputs/manual-actions/` (mark it RESOLVED, blocker cleared)
6. **Never present cached Apollo figures as current.** Live numbers replace what is shown.
7. **Mark previous figures as `Historical Only`** — keep them for audit, labeled and superseded, never displayed as current.
8. **On failure, show the exact live error** returned by the connector, plus which connector was called.

**Read-only guarantee during verification:** never create lists, reveal paid contact data, create sequences, enroll contacts, consume credits, or send messages while checking status. Sequence *capacity* is reported from the existing-sequence count; do **not** test a sequence cap by attempting to create one.

### When Apollo cannot be called (any live-check failure)

Show exactly:

```
Apollo live status: Verification Failed
```

Then show the exact error and the connector id that was called. **Do not** fall back to old values, and **do not** silently substitute the July-22 / free-account figures. The old data stays labeled `Historical Only`; status is `Live Verification Failed` until a live call succeeds again.

### `/abm-status cached` — Apollo output rules

May read saved state and make **no** connector calls, but every Apollo value must display:

- **Last verified:** date + time of the last successful live verification
- **Connector used:** the server id that produced it
- **Data age:** how old the figure is (now − last verified)
- **Status:** `Current` only if the last verification is < 24h old, otherwise `Historical Only`

## Data freshness rules (apply in every mode)

- **Apollo information older than 24 hours must never be shown as current.** If the newest successful Apollo verification is ≥ 24h old, label it `Historical Only` and, in a live/no-arg run, re-verify before reporting.
- Credit balances, plan information, mailbox status, sequence capacity, and connector availability must be **live-verified** (a successful live check < 24h old) before any of the following gated actions — this skill only *reports* readiness; the gate is enforced by `/abm-daily`, `/abm-execute`, and Door-Keeper:
  - Prospecting
  - Revealing contact information
  - Creating lists
  - Creating sequences
  - Enrolling contacts
  - Starting a campaign
- In the report, state plainly whether Apollo is **currently gated-action-ready** (live-verified < 24h) or **not ready — re-run `/abm-status live`**.

## Other connectors (live-by-default, read-only)

- **Monday CRM** — ABM prospects sync (contacts synced vs. pending), board activity.
- **Google Drive** — prospect-review sheet / campaign folder recency.
- **Mixmax** — sequence status + recipients + today's account-wide send count.

For each: attempt a read-only live check in `live`/no-arg mode; if it errors or is unauthorized, report that plainly and fall back to cached **with a clear cached label + last verification time**. Never fabricate.

## Startup

1. Identify the active campaign(s) from `campaigns/*/campaign.yaml` (`status: active`) and `state/active-campaigns.yaml`.
2. Read local state: `state/action-history.json`, `state/contact-history.json`, `state/signal-history.json`, `state/pending-approvals.yaml`, `state/permission-ledger.yaml`, and the latest `outputs/dashboard/dashboard-data.json` / `latest-eod-summary.md`.
3. Resolve the mode (live / cached / no-arg), then run the connector checks the mode calls for.

## Report exactly these fields

- Active campaign(s)
- Current daily run status (from `state/active-campaigns.yaml`)
- **Apollo — connector used, verification timestamp, live vs. cached, plan (Paid? evidence), credits (per category + reset cycle), connected mailbox(es), existing sequences + capacity note, gated-action-ready Yes/No.** On failure: `Apollo live status: Verification Failed` + exact error + connector id.
- Apollo sequence status (for this campaign specifically)
- Mixmax status (sequence + recipients + today's send count) — with live/cached label
- Google Drive review-sheet status — with live/cached label
- Monday synchronization status (contacts synced vs. pending) — with live/cached label
- Contacts waiting for review · Contacts approved · Contacts awaiting sync · Contacts already in sequences
- Pending approvals (from `state/pending-approvals.yaml` + `outputs/approval-queue/`)
- Active Door-Keeper authorizations · Expired authorizations (from `state/permission-ledger.yaml`, compared to today's date)
- Connector blockers · Chrome blockers · Manual actions (from `outputs/manual-actions/`)
- Last EOD report (path + date)
- **Recommended next command** (e.g. `/abm-status live` to refresh a stale check, `/abm-daily`, `/abm-execute today`, or "resolve <blocker> first")

## Rules

- Label every value that can't be confirmed: `Not Available` / `Not Connected` / `Not Tracked` / `Awaiting Sync`.
- Treat an expired permission-ledger authorization as **expired**, never as active.
- Do not call any write tool against an external system. If asked to "just sync while you're there," decline and point to `/abm-execute`. Local state/dashboard updates that record a live check are the only writes allowed.
- Do **not** perform outreach in any mode.
- This behavior is the Campaign Director's; it does not invoke worker subagents (a read-only status check doesn't need them).
- After a successful live paid-account verification, remove/supersede the old July-22 free-account Apollo values from state and the dashboard — keep them only as `Historical Only`, never as current.
