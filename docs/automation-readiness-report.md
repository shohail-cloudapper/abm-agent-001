# Automation Readiness Report

How ready each piece of the daily ABM workflow is to run without a human in the loop, based on what's actually connected today (see `docs/tool-and-connector-inventory.md`).

## Per-task classification

| Task | Classification | Why |
|---|---|---|
| Signal research via ZoomInfo enrichment | **Fully automatable** | Connected, read-only, no external-facing action. |
| Signal research via web search | **Fully automatable** | No auth needed. |
| Signal/conversation research on LinkedIn, Reddit, communities | **Automatable with one-time permission** | Needs a one-time Claude in Chrome per-site authorization; after that, read-only browsing needs no further approval. |
| Contact discovery via ZoomInfo | **Fully automatable** | Connected, read-only. |
| Contact discovery via LinkedIn/Sales Navigator | **Automatable with one-time permission** | Same Chrome per-site authorization as above; read-only. |
| Data cleaning, dedup, suppression checks | **Fully automatable** | Local state only. |
| Lead scoring | **Fully automatable** | Local state only. |
| Content drafting (email sequences, comments, talking points) | **Fully automatable** | Drafting has no external side effect; always lands as `PENDING_APPROVAL`. |
| Gmail draft creation | **Fully automatable** | The connected Gmail tool has no send capability at all — creating a draft is as far as it can go regardless of settings. |
| Mixmax sequence enrollment | **Automatable with batch approval** | External and reaches real inboxes — needs a standing batch/campaign authorization first. |
| LinkedIn invitations | **Requires recurring human review** | Beyond needing authorization, LinkedIn account-safety risk (rate limits, warnings) makes this worth a human glance even under a broader approval mode, at least until the campaign has a track record. |
| Comment / community response publishing | **Requires recurring human review** | Public and reputational — recommend keeping this at per-batch or per-item review even if other channels move to `approved_automation`. |
| External CRM record creation | **Automatable with batch approval** | Fine to batch-approve once a campaign is established. |
| External CRM stage changes | **Requires recurring human review** | Stage changes affect pipeline reporting and ownership — worth a human check even under `approved_automation`. |
| Reply / engagement monitoring (Gmail, Mixmax, LinkedIn reads) | **Fully automatable** | Read-only. |
| Reply classification | **Fully automatable** | Local reasoning over already-fetched content. |
| Stop-condition enforcement (opt-out, complaint, negative reply) | **Fully automatable, and mandatory** | This is a safety default — it must never wait for approval. |
| Daily report generation | **Fully automatable** | Reads state, writes a file. |
| Prospecting from a source with no connector (e.g. TeamBlind) | **Manual because no connector or browser access exists** | Routed through `workflows/manual-prospecting-workflow.md`. |
| Anything on Google Sheets (row-level) | **Manual until a Sheets connector is authorized** | Not connected this session — see the gap noted in the tool inventory. |
| Apollo / HubSpot / Klaviyo / Notion / Slack / Close / Outreach.io / Fireflies / Clay / Canva / Figma / Ahrefs / Amplitude / Supermetrics / Similarweb-based tasks | **Blocked by missing authorization** | All show as "Requires user permission" — not usable until you authorize them. |

## Chrome / browser specifics

- **Sites that need Claude in Chrome permission the first time they're touched:** LinkedIn (incl. Sales Navigator), any gated community or forum a campaign approves, and any web-based CRM UI you choose as a `crm_destination`.
- **Sites to set "Always allow" on only after you've approved them once:** LinkedIn and any web-based CRM — these are exactly the surfaces where write-capable actions (invites, messages, CRM edits) eventually happen, so don't blanket-approve them before seeing how the campaign behaves.
- **TeamBlind:** treat as manual regardless of Chrome access — it's not a reliable automation target.

## Scheduling

- **Can be scheduled to run unattended:** signal research, contact discovery/refresh, data cleaning, lead scoring, content drafting, Gmail draft creation, reply/engagement monitoring, stop-condition enforcement, and daily report generation. The connected Scheduled Tasks capability is the mechanism for this — not turned on by this setup pass.
- **Needs an active Claude in Chrome / browser session available:** anything touching LinkedIn, gated communities, or a web-based CRM. A scheduled run that hits these steps needs the browser connection live at run time, not just the schedule firing.
- **Unavoidable confirmations that can't be scripted around:** Chrome's own first-visit per-site permission prompt, and any 2FA/security-verification challenge a platform presents. These always get surfaced to you live — never worked around.

## Missing integrations (recap)

Google Sheets (row-level), Apollo, HubSpot, Klaviyo, Notion, Slack, Canva, Figma, Ahrefs, Amplitude, Supermetrics, Similarweb, Close, Outreach.io, Fireflies, Clay. None of these are assumed available anywhere in the agent instructions — see `docs/tool-and-connector-inventory.md` for the full breakdown.

## Recommended first campaign for testing

Keep the pilot small and reversible: **one target-account list of 5–10 accounts, `approval_mode: drafts_only`, and only the connectors already authorized** — ZoomInfo enrichment, web research, and Gmail drafts. Leave LinkedIn invitations and Mixmax enrollment switched off (or at `batch_approval` with a tiny volume cap) until you've watched a few days of signal quality and draft quality first. Move to `batch_approval` or `campaign_preapproved` only once the daily report shows the pipeline behaving the way you expect.
