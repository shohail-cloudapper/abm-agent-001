# Skill Inventory

Audit of every existing, reusable Claude Skill found for this system. Source of truth for all 16 skills:

- 15 skills as polished source files at `C:\Users\shohail\Documents\Claude\SKILLS\Skills\*.md`, each also packaged as a zip in `C:\Users\shohail\Documents\Claude\SKILLS\psyops_15_role_skill_zips\`.
- 1 additional skill, `manual-lead-capture`, in its own folder at `C:\Users\shohail\Documents\Claude\SKILLS\manual-lead-capture\Skill.md`.

None of these files were changed, renamed, merged, or deleted. All 16 are confirmed installed and callable in this environment (they appear in the session's available-skills listing under the `anthropic-skills:` namespace).

No skill was found broken, so no new skill was created — including Door-Keeper, which already exists under its exact name `door-keeper` and is reused as-is for the permission system.

---

## Governance & shared skills

### door-keeper
- **File path:** `Documents/Claude/SKILLS/Skills/door-keeper.md`
- **Purpose:** Pre-flight authorization gate. Checks access, login state, credits, rate limits, and platform warnings before any agent touches a platform or takes an external action. Returns `ALLOWED` / `ALLOWED_WITH_APPROVAL` / `LIMITED` / `BLOCKED`.
- **Inputs:** platform or action name, active campaign config, current access/credit/limit state.
- **Outputs:** access decision, checked limits, safe volume, restriction reason, required human action.
- **Required data:** campaign's approved-platform list, prior errors/warnings.
- **Required tools:** none of its own — it reasons over whatever access state the calling agent supplies.
- **Agents that use it:** Campaign Director (governance owner) and Outreach, Engagement & Follow-Up Agent (enforces it before every external action); consulted by Signal and Buying Committee agents before browser research.
- **Supports browser actions:** No (it's a decision skill, not an executor).
- **Read or write:** Read-only reasoning; writes its decision to the record via `record-keeper`.
- **Human approval required:** This skill is *how* approval requirements get enforced — it decides when approval is needed, it doesn't grant it.
- **Overlap / risk:** None. This is the single skill this whole system's safety model is built around — see `workflows/approval-workflow.md` and `state/permission-ledger.yaml`.

### record-keeper
- **File path:** `Documents/Claude/SKILLS/Skills/record-keeper.md`
- **Purpose:** Single system-of-record writer. Every other skill hands its result here after a decision, approval, external action, or status change.
- **Inputs:** action/result to save, campaign's system-of-record schema, prior record version, stable IDs.
- **Outputs:** write confirmation, previous/new status, fields updated, record link.
- **Required data:** campaign's declared CRM/database/Sheet destination.
- **Required tools:** whatever connector the campaign's `crm_destination` points to (today: local `state/*.json`/`*.yaml` files; later, Monday.com or Google Sheets once wired).
- **Agents that use it:** all five agents.
- **Supports browser actions:** No.
- **Read or write:** Read (current record) + Write (append history, never overwrite silently).
- **Human approval required:** No — logging completed/approved work is not itself an external action.
- **Overlap / risk:** None.

### fact-checker
- **File path:** `Documents/Claude/SKILLS/Skills/fact-checker.md`
- **Purpose:** Verifies claims, dates, quotes, and statistics before they're used in a signal writeup, a draft, or a public comment.
- **Inputs:** the claim/draft, required freshness, approved source hierarchy.
- **Outputs:** verdict (`VERIFIED` / `SUPPORTED_WITH_QUALIFIER` / `OUTDATED` / `CONFLICTING` / `UNVERIFIED` / `FALSE`), approved wording, source + date.
- **Required data:** campaign's approved-source hierarchy.
- **Required tools:** web search (`WebSearch`/`WebFetch`).
- **Agents that use it:** Personalization & Campaign Content Agent (primary, every draft); Account Signal & Intelligence Agent (verifying claims inside a signal); Outreach Agent (verifying claims in a follow-up); Campaign Director (messaging governance spot-checks).
- **Supports browser actions:** Only ordinary web search, not authenticated/logged-in browsing.
- **Read or write:** Read-only (research).
- **Human approval required:** No — this *prevents* bad claims from reaching a human approval step, it isn't itself gated.
- **Overlap / risk:** None.

### scoreboard-reporter
- **File path:** `Documents/Claude/SKILLS/Skills/scoreboard-reporter.md`
- **Purpose:** Turns the system-of-record into the daily/weekly report — separates real activity from real outcomes, flags bottlenecks.
- **Inputs:** campaign, period, goals/targets, prior-period baseline.
- **Outputs:** activity/quality/response/pipeline metrics, blockers, recommended actions.
- **Required data:** campaign's system-of-record data for the period.
- **Required tools:** none beyond reading state.
- **Agents that use it:** Campaign Director (primary, daily report); Outreach Agent (secondary, engagement-specific stats).
- **Supports browser actions:** No.
- **Read or write:** Read-only.
- **Human approval required:** No.
- **Overlap / risk:** Partially overlaps with the pre-existing, hardcoded `data-studio-analytics---psyopsshohail` scheduled task (fixed Looker Studio URL, fixed "PsyOpsShohail" tag). Recommend retiring that one-off automation in favor of this reusable skill once GA/Looker Studio is wired into a campaign's `data_sources`. Not touched in this pass.

---

## Account Signal & Intelligence Agent skills

### conversation-radar
- **File path:** `Documents/Claude/SKILLS/Skills/conversation-radar.md`
- **Purpose:** Finds relevant public conversations (LinkedIn, Reddit, Quora, X, UKG-style communities, forums) and recommends `COMMENT` / `MONITOR` / `RESEARCH_AUTHOR` / `RESEARCH_COMPANY` / `IGNORE`.
- **Inputs:** campaign keywords/ICP/personas, target opportunity count, lookback window, already-engaged URLs.
- **Outputs:** scored opportunity list — platform, URL, author, date, signal type, relevance/urgency, recommended action.
- **Required data:** campaign's approved platforms and identities.
- **Required tools:** web search; browser automation (Claude in Chrome or the in-app Browser) for platforms with no API — LinkedIn, most communities.
- **Agents that use it:** Account Signal & Intelligence Agent (primary).
- **Supports browser actions:** Yes — this is its main mode for LinkedIn/community discovery.
- **Read or write:** Read-only.
- **Human approval required:** No for the research itself; any comment it recommends goes to `comment-buddy` and then human approval before publishing.
- **Overlap / risk:** None as a skill. Note: legacy scheduled task `linkedin-connection-sent` tracks a different thing (connections sent, not conversations found) — no real overlap.

### company-detective
- **File path:** `Documents/Claude/SKILLS/Skills/company-detective.md`
- **Purpose:** Builds a sourced account profile — industry, size, locations, tech/operational signals, campaign triggers — and a fit/signal score.
- **Inputs:** company name/domain, campaign ICP, existing CRM history.
- **Outputs:** company brief, `account_fit_score`, `signal_score`, evidence links, recommendation.
- **Required data:** campaign ICP, target industries/geographies, exclusion list.
- **Required tools:** web search; ZoomInfo enrichment tools once that connector is authorized for the campaign (`enrich_companies`, `enrich_news`, `enrich_scoops`, `account_research`).
- **Agents that use it:** Account Signal & Intelligence Agent (primary); consulted by Account & Buying Committee Agent before contact search.
- **Supports browser actions:** Yes, for sources with no API (company sites, press pages).
- **Read or write:** Read-only.
- **Human approval required:** No.
- **Overlap / risk:** Conceptually overlaps with the connected `zoominfo:account-research` skill. Not a duplicate to remove — recommend `company-detective` calls ZoomInfo as its evidence source when a campaign's `data_sources` includes ZoomInfo, and falls back to plain web research otherwise.

---

## Account & Buying Committee Agent skills

### people-finder
- **File path:** `Documents/Claude/SKILLS/Skills/people-finder.md`
- **Purpose:** Builds the smallest useful buying committee — decision-makers, champions, evaluators, blockers — without over-collecting personal data.
- **Inputs:** qualified company record, target roles/seniority/geography/exclusions.
- **Outputs:** ranked contacts with `committee_role`, `role_relevance_score`, `data_confidence`, recommended channel.
- **Required data:** campaign persona map, existing CRM contacts, outreach history.
- **Required tools:** browser automation (LinkedIn, Sales Navigator, company team pages); ZoomInfo `search_contacts` / `get_recommended_contacts` once authorized.
- **Agents that use it:** Account & Buying Committee Agent (primary).
- **Supports browser actions:** Yes.
- **Read or write:** Read-only.
- **Human approval required:** No for research; the contacts it surfaces still pass through suppression/cooldown checks before any outreach agent may act on them.
- **Overlap / risk:** Overlaps conceptually with the connected `zoominfo:buying-committee` skill — same recommendation as `company-detective` above (use as evidence source, don't duplicate logic).

### data-cleaner
- **File path:** `Documents/Claude/SKILLS/Skills/data-cleaner.md`
- **Purpose:** Normalizes raw prospect records and checks duplicates, suppression, customers, partners, and prior outreach before anything is qualified.
- **Inputs:** raw records (file/sheet/pasted), campaign schema, suppression list.
- **Outputs:** `CLEAN` / `NEEDS_REVIEW` / `SUPPRESSED` / `DUPLICATE` / `INVALID` per record, with normalization changes and duplicate links.
- **Required data:** campaign's suppression/customer/partner/opportunity lists.
- **Required tools:** the campaign's CRM/DB/Sheet, read access at minimum.
- **Agents that use it:** Account & Buying Committee Agent (primary).
- **Supports browser actions:** No.
- **Read or write:** Read (source records) + Write (clean record output) to the campaign's own data store, not an external CRM.
- **Human approval required:** No.
- **Overlap / risk:** None; works directly upstream of `lead-judge` and downstream of `manual-lead-capture`.

### lead-judge
- **File path:** `Documents/Claude/SKILLS/Skills/lead-judge.md`
- **Purpose:** Scores fit, intent, timing, role relevance, and data confidence into a tier (`A`/`B`/`C`/`REJECTED`) using campaign-defined weights.
- **Inputs:** clean company + contact records, campaign scoring weights and exclusions.
- **Outputs:** component scores, `final_score`, `tier`, `qualification_reason`, recommended channel/message angle.
- **Required data:** campaign's scoring model, suppression/customer/opportunity status.
- **Required tools:** the campaign's CRM/DB.
- **Agents that use it:** Account & Buying Committee Agent (primary, qualification gate before handoff to content).
- **Supports browser actions:** No.
- **Read or write:** Read-only.
- **Human approval required:** No.
- **Overlap / risk:** Conceptually parallels the connected `zoominfo:score-accounts` / `zoominfo:score-leads` skills — recommend `lead-judge` treat a ZoomInfo score as one input signal when available, not a replacement for the campaign-specific weighting.

### manual-lead-capture
- **File path:** `Documents/Claude/SKILLS/manual-lead-capture/Skill.md`
- **Purpose:** Fallback capture for any lead source with no API/connector — asks for source type, location, destination format, required fields, and whether email is required. Never scrapes, exports, or bypasses access controls.
- **Inputs:** source type/link/file/location, destination format, required fields.
- **Outputs:** structured spreadsheet/table of captured leads with full provenance.
- **Required data:** none beyond what's visible at the named source.
- **Required tools:** whatever destination the user names (a Sheet, CSV, or Drive file); browser automation if the source is a website that must be read manually.
- **Agents that use it:** Account & Buying Committee Agent (primary fallback); occasionally Account Signal & Intelligence Agent when a raw signal source has no connector.
- **Supports browser actions:** Yes, read-only page viewing only — never scraping.
- **Read or write:** Read (source) + Write (destination table).
- **Human approval required:** No for capture itself; captured records still go through `data-cleaner` and suppression checks before qualification.
- **Overlap / risk:** ⚠️ **Needs review** — the file at its documented path is the creation brief originally used to generate the skill, not a finished skill in the frontmatter/workflow/output-contract shape the other 15 use. It is confirmed installed and functions correctly (it's callable by name in this session), so nothing was changed; regenerating its local source to match the other 15's format is a good housekeeping item for later, not a blocker.

---

## Personalization & Campaign Content Agent skills

### email-writer
- **File path:** `Documents/Claude/SKILLS/Skills/email-writer.md`
- **Purpose:** Writes the full outreach sequence — subject options plus progressive follow-ups — from a qualified prospect's verified signal and persona.
- **Inputs:** qualified prospect/segment, campaign messaging/claims/CTA, sequence length.
- **Outputs:** subject options, `sequence_steps`, CTA, `claims_used`.
- **Required data:** campaign messaging framework, proof points, prohibited language.
- **Required tools:** `fact-checker` for any claim.
- **Agents that use it:** Personalization & Campaign Content Agent (primary).
- **Supports browser actions:** No.
- **Read or write:** Read-only (drafting).
- **Human approval required:** Yes — every sequence is `PENDING_APPROVAL`/`NEEDS_REVIEW` output; this skill never sends.
- **Overlap / risk:** Light conceptual overlap with the connected `zoominfo:personalize-email` skill (single-touch variant generator). Keep `email-writer` as the sequence author of record; optionally borrow angle ideas from ZoomInfo's version.

### comment-buddy
- **File path:** `Documents/Claude/SKILLS/Skills/comment-buddy.md`
- **Purpose:** Drafts one primary and up to two alternative social comments from a Conversation Radar opportunity, scored for promotional risk.
- **Inputs:** post/opportunity record with context, approved identity.
- **Outputs:** draft(s), `promotional_risk`, `approval_status`.
- **Required data:** campaign voice rules, prior engagement history.
- **Required tools:** browser automation for the actual platform (to read full context); `fact-checker` for factual claims.
- **Agents that use it:** Personalization & Campaign Content Agent (drafting, primary); Outreach, Engagement & Follow-Up Agent (publishing the approved draft, secondary).
- **Supports browser actions:** Yes, for reading the post; publishing is a separate, approval-gated step performed by the Outreach agent.
- **Read or write:** Read (research) + local draft write; the actual public "write" (posting) only happens after approval, executed by the Outreach agent.
- **Human approval required:** Yes, always, before anything is published.
- **Overlap / risk:** None — the split between "draft" (Personalization) and "publish" (Outreach) is intentional and is spelled out in both agent files so responsibility doesn't blur.

---

## Outreach, Engagement & Follow-Up Agent skills

### draft-assistant
- **File path:** `Documents/Claude/SKILLS/Skills/draft-assistant.md`
- **Purpose:** Creates reviewable Gmail drafts for strategic, threaded, referral, or sensitive outreach. Never sends.
- **Inputs:** recipient/thread, purpose, verified context.
- **Outputs:** Gmail draft link, status `PENDING_APPROVAL`.
- **Required data:** thread history, recipient relationship.
- **Required tools:** Gmail connector's `create_draft` tool.
- **Agents that use it:** Outreach, Engagement & Follow-Up Agent (primary).
- **Supports browser actions:** No — uses the Gmail MCP connector directly, not browser automation.
- **Read or write:** Write (creates a draft — not a send).
- **Human approval required:** Yes for send; drafting itself needs no approval since nothing external is dispatched.
- **Overlap / risk:** None. Note: the currently connected Gmail tool set has **no send tool at all** — so this skill's "never send" rule is enforced by the platform connection itself, not only by instruction.

### sequence-sender
- **File path:** `Documents/Claude/SKILLS/Skills/sequence-sender.md`
- **Purpose:** Enrolls only approved, unsuppressed contacts into a named Mixmax sequence, dry-running the eligible list first.
- **Inputs:** approved records, exact sequence name/version, daily limit.
- **Outputs:** eligible/enrolled/excluded counts, per-contact result.
- **Required data:** campaign's approved sequence, sending account.
- **Required tools:** Mixmax connector (sequences).
- **Agents that use it:** Outreach, Engagement & Follow-Up Agent (primary).
- **Supports browser actions:** No — uses the Mixmax MCP connector.
- **Read or write:** Write (enrollment is an external action).
- **Human approval required:** Yes, per campaign's approval mode, before enrollment — this skill produces a dry-run list first specifically so approval can happen before anything is live.
- **Overlap / risk:** None.

### connection-helper
- **File path:** `Documents/Claude/SKILLS/Skills/connection-helper.md`
- **Purpose:** Sends approved LinkedIn invitations only, after checking identity, daily limits, suppression, and duplicate/pending status.
- **Inputs:** approved records, identity, platform/campaign limits.
- **Outputs:** sent/excluded counts, remaining capacity.
- **Required data:** campaign's LinkedIn rules and identity.
- **Required tools:** browser automation (Claude in Chrome / in-app Browser) — there is no LinkedIn API connector.
- **Agents that use it:** Outreach, Engagement & Follow-Up Agent (primary).
- **Supports browser actions:** Yes — this is its only mode of execution.
- **Read or write:** Write (an actual LinkedIn invitation is an external action).
- **Human approval required:** Yes, always, per `workflows/browser-action-policy.md` — this is explicitly one of the actions that always requires Door-Keeper authorization.
- **Overlap / risk:** None.

### follow-up-buddy
- **File path:** `Documents/Claude/SKILLS/Skills/follow-up-buddy.md`
- **Purpose:** Classifies what happened after outreach (accepted, replied, objection, referral, opt-out, meeting booked, etc.) and prepares the one correct next step.
- **Inputs:** reply/accepted-connection record, full conversation history.
- **Outputs:** `response_class`, conflicting actions stopped, `next_due_at`, owner.
- **Required data:** campaign follow-up rules, stop conditions.
- **Required tools:** Gmail, Mixmax, browser (LinkedIn).
- **Agents that use it:** Outreach, Engagement & Follow-Up Agent (primary).
- **Supports browser actions:** Yes, for LinkedIn reply/acceptance checks.
- **Read or write:** Read (classification) + Write (stops conflicting sequences, which is itself a safety action, not a new outreach).
- **Human approval required:** No for stopping/pausing outreach (that's a mandatory safety action, not a new external send); yes for any new reply it drafts.
- **Overlap / risk:** This is where the old Projects design's imagined "Reply Classifier" skill actually lives — see the note below. No separate skill needed.

---

## Cross-cutting findings

1. **All 16 skills are already campaign-agnostic.** Every one takes a `$campaign` argument and refers only to "the campaign's declared CRM, database, or Google Sheet," never a hardcoded product or account list. No skill needed to be rewritten to meet the reusability requirement.
2. **"Reply Classifier" and "Create Handoff"** — referenced constantly in an older, now-retired Claude-Projects-based design for this same campaign work — were never built as standalone skills. Reply classification already lives inside `follow-up-buddy`; "handoff" is just the `next_agent` / `next_action` / `due_at` fields every skill already returns in its output contract. The new Campaign Director owns routing directly. No new skill was created for either.
3. **No skill was broken, unclear, or missing** to the point of needing a replacement — including Door-Keeper, which exists exactly as required.
4. **manual-lead-capture** is flagged for a documentation cleanup (see above) but was not modified.
