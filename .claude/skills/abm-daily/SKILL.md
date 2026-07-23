---
name: abm-daily
description: Run the daily ABM research-and-preparation cycle for the active campaign — scout, verify, store to the Drive review sheet, draft Apollo/Mixmax/LinkedIn content, and present a decision package. Prepares only; stops before any external write. Use when the user types /abm-daily [people-limit] [email-platform] [linkedin-connections] [linkedin-comments].
argument-hint: "[people-limit] [email-platform: apollo|mixmax|pending|none] [linkedin-connections] [linkedin-comments]"
arguments:
  - people_limit
  - email_platform
  - linkedin_connections
  - linkedin_comments
user-invocable: true
disable-model-invocation: false
effort: high
---

# /abm-daily

Runs the existing five-agent daily workflow (`workflows/daily-abm-workflow.md`) in **prepare-only** mode. Nothing external is written, enrolled, sent, or published. The run ends with a decision package for the owner.

## Arguments (fall back to `campaign.yaml` → `daily_defaults` when omitted)

- `$people_limit` — max **unique** people to evaluate (default 20).
- `$email_platform` — `apollo` | `mixmax` | `pending` | `none` (default `pending_user_selection`).
- `$linkedin_connections` — max LinkedIn connection notes to **prepare** (default 10).
- `$linkedin_comments` — max LinkedIn comments to **prepare** (default 5).

Examples: `/abm-daily` · `/abm-daily 20 pending 10 5` · `/abm-daily 100 apollo 25 10`

## Director planning (read first)

The Campaign Director reads: campaign config, yesterday's EOD report, the Google Drive Prospect Review Sheet, Monday CRM, Apollo, Mixmax, LinkedIn activity, local state, the permission ledger, pending approvals, suppression records, cooldown records, existing sales ownership, and recent connector failures. If a source's connector is unreachable, note it and continue — don't fabricate its contents.

## Scouting (account-signal-intelligence + account-buying-committee)

- Scout **no more than `$people_limit` unique people**. Quality over quantity — do not pad the limit with weak contacts.
- Source priority (from `campaign.yaml` → `source_priority`): 1) Apollo, 2) UKG Community, 3) ZoomInfo, 4) LinkedIn/Sales Navigator, 5) company websites, 6) job listings, 7) public web.
- Access priority: connector → Claude in Chrome → manual action. Never bypass CAPTCHA/login/rate limits.
- **Log every research attempt** to `logs/scouting/YYYY-MM-DD/source-crawl-log.csv` with columns: date_time, agent, skill, platform, domain, exact_url, search_query, account, contact, access_method, connector, chrome_used, records_reviewed, relevant_results, evidence, status, errors, blockers. **Do not record a source as "crawled" when only a search-result snippet was seen** — mark that as `snippet_only`.

## Contact verification (before recommending anyone)

Check across: Drive review data, Monday ABM Campaign Prospects, Monday Sales Contacts/Leads/Opportunities, Apollo, Mixmax, local contact history, business email, LinkedIn URL, full name + company domain, previous outreach, existing sequence membership, previous replies, bounce history, unsubscribe status, suppression, cooldown, record owner, LinkedIn invitation status.

Assign exactly one eligibility status: `new_contact` · `existing_not_contacted` · `previously_contacted_eligible` · `active_in_apollo` · `active_in_mixmax` · `cooldown_active` · `replied_previously` · `bounced` · `unsubscribed` · `suppressed` · `duplicate` · `owned_by_another_person` · `excluded_account` · `insufficient_information`.

## Contact storage (account-buying-committee)

Save every newly researched contact **first** to the Google Drive Prospect Review Sheet with: Review Decision = `Pending Review`, Monday Sync Status = `Not Approved`, Selected Outreach Platform = the requested platform, Apollo Status = `Prepared Only`, Mixmax Status = `Prepared Only`, LinkedIn Status = `Prepared Only`. **Do not sync to Monday. Do not enroll in Apollo/Mixmax.**

## Content preparation (personalization-campaign-content)

For every eligible person prepare **both** Apollo-ready and Mixmax-ready versions: personalized subject, Email 1, Follow-up 1, proof-point email, practical follow-up, respectful final email, recommended delays, supporting asset, CTA, personalization rationale, Apollo field mapping, Mixmax field mapping. Ground every message in a **real** account signal, the person's actual role, a relevant operational issue, and approved CloudApper positioning (run `anthropic-skills:cloudapper-messaging` + `fact-checker`). Never invent signals/claims/dates/proof-points/pain-points. Never position CloudApper as a UKG replacement.

## LinkedIn preparation (personalization-campaign-content)

- Prepare ≤ `$linkedin_connections` connection notes; check existing connection, pending/previous invitation, previous messages, suppression, ownership, relevance.
- Prepare ≤ `$linkedin_comments` comments, prioritizing: 1) posts from target contacts, 2) posts from their companies, 3) relevant hrPad/UKG/WFM/employee-self-service/timekeeping/frontline/clock-modernization posts. Each comment: based on the actual post, helpful, natural, unique, non-promotional, in Shohail's voice.
- **Do not send invitations or publish comments during `/abm-daily`.**

## Daily decision package (Director presents)

Contacts researched · eligible · waiting for review · duplicates · existing sales-owned · suppressed · needs-more-research · Apollo sequence readiness · Mixmax sequence readiness · Apollo credit estimate · personalized email summary · LinkedIn connection requests · LinkedIn posts + comment drafts · proposed Monday changes · risks · required authorization.

If `$email_platform` is `pending`, **ask the owner to choose Apollo / Mixmax / No Email Outreach.** Then stop — the next step is `/abm-execute` after the owner approves.

## Hard stop

This skill stops before every external write. It creates/updates a consolidated approval request in `outputs/approval-queue/` and mirrors it into `state/pending-approvals.yaml`. Execution is a separate, explicit `/abm-execute`.
