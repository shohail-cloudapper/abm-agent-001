# Tool & Connector Inventory

Snapshot taken during setup of this system. **Re-check this file whenever a campaign is onboarded** — connector availability changes as you authorize new tools, and agents must never claim access to something that isn't actually connected at run time.

Classification key: **Available (read-only)** · **Available (write-capable)** · **Available via Claude in Chrome** · **Requires user permission** · **Not connected** · **Manual process required**

---

## Research & enrichment

| Capability | Status | Notes |
|---|---|---|
| ZoomInfo (account/contact/intent/news/scoop enrichment) | **Available (read-only)** | Company + contact search, enrichment, intent, news, scoops, similar-company/contact lookup, GTM context. Also exposes matching `zoominfo:*` skills (account-research, buying-committee, score-accounts, score-leads, personalize-email). No write actions — pure research. |
| Apollo | **Available (write-capable)** | LIVE-VERIFIED 2026-07-22T07:15:33Z via MCP server `b4eb3cd7-8008-4759-8812-16897005eddd` — authorized paid account (Shohail Tamim, team 667e9f96900c1e03f757a8cb). 14/14 read-only verification calls succeeded: profile, credit stats, mailboxes, sequences, schedules. Read (people/account search, enrichment, engagement) confirmed; write actions (list/sequence create, enrollment) exist but are Door-Keeper-gated. A separate `plugin:sales:apollo` connector is an **unauthenticated duplicate — do not use**. |
| ZoomInfo (plugin variant) | Requires user permission | A second, plugin-based ZoomInfo connector is also listed as pending auth — likely redundant with the one already connected; don't authorize both without checking for duplication. |
| Clay | Requires user permission | Not yet authorized. |
| Similarweb | Requires user permission | Not yet authorized. |
| Ahrefs | Requires user permission | Not yet authorized. |

## Web, social & community research

| Capability | Status | Notes |
|---|---|---|
| General web search | **Available (read-only)** | `WebSearch` / `WebFetch`, no auth needed. |
| ScrapeGraphAI structured extraction (local) | **Available (read-only)** | Installed 2026-11-23: `scrapegraphai` 2.3.1 + Playwright Chromium + Ollama 0.40.16 with `llama3.2`. Run through `tools/scrapegraph/scrape_page.py`, used by `account-signal-intelligence`. Public, non-login pages only. It refuses LinkedIn/UKG Community/TeamBlind and other gated sites, and anything robots.txt disallows. Local model only, so no page data leaves the machine. Output is unverified: confirm against the source before using it as a signal. Needs the Ollama app running; if it isn't, the helper exits with code 2 and the agent writes a manual action. |
| LinkedIn (including Sales Navigator) | **Available via Claude in Chrome** | No API connector exists. Read (search, view posts/profiles) works through the user's already-logged-in Chrome session. Any write action (invite, message, comment) is Door-Keeper-gated regardless of channel. |
| Reddit, Quora, X, UKG Community, other forums | **Available via Claude in Chrome** | Same pattern — no API, browser-only, read is fine, posting is gated. |
| TeamBlind | Manual process required | Anonymous/verification-gated community; browser automation is unreliable here. Treat any TeamBlind finding as a `manual-lead-capture`-style entry, sourced by the human. |

## CRM / database / system of record

| Capability | Status | Notes |
|---|---|---|
| Monday.com (boards, items, columns, automations, docs, workflows) | **Available (write-capable)** | Full read/write CRM-style surface — strongest current candidate for a campaign's `crm_destination`. Write actions here (`create_item`, `change_item_column_values`, etc.) still require Door-Keeper authorization when they represent an external-facing or stage-changing update. |
| Google Sheets (row-level API) | **Not connected** | No dedicated Sheets connector is present this session, only Google Drive file operations (below). The earlier "Command Center" design leaned on Sheets entirely — this is a real gap. Until resolved, use Monday.com or Drive-hosted files as the system of record, or ask the user to authorize a Sheets connector. |
| Google Drive (file read/write, search, metadata) | **Available (write-capable)** | Can read and create files, including exported Sheets, but has no row-level Sheets editing. |
| Atlassian (Jira/Confluence) | **Available (write-capable)** | Connected, but non-core to ABM — only relevant if a campaign explicitly tracks work in Jira. |
| HubSpot | Requires user permission | Not yet authorized. |
| Close | Requires user permission | Not yet authorized. |
| Monday.com (plugin variant) | Requires user permission | A second, plugin-based Monday connector is pending auth — likely redundant with the one already connected. |

## Email & sequencing

| Capability | Status | Notes |
|---|---|---|
| Gmail | **Available (write-capable, draft-only)** | `create_draft`, `list_drafts`, `search_threads`, labeling, thread/message read. **No send tool exists at all** — this is a hard platform-level guarantee that `draft-assistant`'s "never send" rule can't be bypassed by this system. |
| Mixmax (sequences, meetings) | **Available (write-capable)** | Sequence creation/editing and meeting tools connected. `sequence-sender` uses this; enrollment is still Door-Keeper-gated. |
| Outreach.io | Requires user permission | Not yet authorized. |
| Klaviyo | Requires user permission | Not yet authorized. |

## Browser automation

| Capability | Status | Notes |
|---|---|---|
| In-app Browser pane | **Available** | Already active this session, no extra auth step. Good for general research where a logged-in personal session isn't required. |
| Claude in Chrome (real Chrome, user's logged-in sessions) | **Available via Claude in Chrome** | Tool schemas are deferred (loaded on demand via `ToolSearch`) but the capability itself is present. This is the right surface for anything that needs the user's actual LinkedIn/UKG Community/CRM login. Per-site permission prompts happen the first time a new origin is touched — those prompts must be surfaced to the user, never auto-dismissed. |

## Collaboration / notification

| Capability | Status | Notes |
|---|---|---|
| Slack | Requires user permission | Not yet authorized. Relevant only if a campaign wants Slack-based approval notifications. |
| Notion | Requires user permission | Not yet authorized. |
| Fireflies (call transcripts) | Requires user permission | Not yet authorized. Would feed `follow-up-buddy` if a campaign records sales calls. |

## Design / analytics (marketing-plugin tools, mostly out of scope for daily ABM ops)

| Capability | Status | Notes |
|---|---|---|
| Canva, Figma | Requires user permission | Only relevant if Personalization agent needs to generate visual assets — not part of the default daily loop. |
| Amplitude, Supermetrics | Requires user permission | Would feed signal/reporting if a campaign tracks product usage or cross-channel spend. |

## Scheduling & session infrastructure

| Capability | Status | Notes |
|---|---|---|
| Scheduled Tasks connector | **Available (write-capable)** | Can create/list/update/delete a recurring scheduled task — this is the mechanism for "run the daily workflow automatically." Used to wire up `workflows/daily-abm-workflow.md` once a campaign goes live; not triggered automatically by this setup pass. |

---

## Pre-existing one-off automations found (not part of this system, not touched)

Four hardcoded scheduled tasks already exist in `Documents/Claude/Scheduled/` — `daily-inbox-reply-digest`, `linkedin-connection-sent`, `data-studio-analytics---psyopsshohail`, `gchat-mentioned-summary`. Each is hardcoded to one email address, one report URL, or one tracking tag, and each overlaps with a piece of this system (`follow-up-buddy`, `connection-helper`'s tracking, `scoreboard-reporter`). They were left running as-is. Recommended follow-up once the daily workflow here is live: fold their function into a campaign's `campaign.yaml` and retire the standalone versions, so metrics aren't computed twice in two different places.

## Bottom line for agent instructions

- Never assume Google Sheets, Apollo, HubSpot, Slack, Notion, or any "Requires user permission" row is reachable — check this file (or its refreshed equivalent) at the start of a run, and downgrade to a manual action in `outputs/manual-actions/` when a required connector isn't actually connected.
- Gmail send, LinkedIn invite/message/comment, and any CRM stage change are **never** one-click actions in this system regardless of connector status — those are Door-Keeper-gated by design, not by accident of missing integration.
