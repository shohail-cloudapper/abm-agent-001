# ABM Agent Team

A reusable, multi-campaign Account-Based Marketing operating system. One set of agents and Skills; every campaign supplies its own configuration instead of getting its own agent team.

## What this is

```
Main Claude Code session (acts as ABM Campaign Director — see CLAUDE.md)
   ├─▶ account-signal-intelligence       — finds and scores account signals
   ├─▶ account-buying-committee          — finds, verifies, and qualifies contacts
   ├─▶ personalization-campaign-content  — drafts personalized content (never sends)
   └─▶ outreach-engagement-followup     — prepares/executes approved actions, tracks engagement
```

Claude Code subagents can't invoke other subagents, so the Director's coordination role lives in the main session (driven by `CLAUDE.md`), not as a fifth subagent in the delegation chain. `.claude/agents/abm-campaign-director.md` still exists as a complete, valid agent definition for portability.

## Folder map

| Path | What's in it |
|---|---|
| `CLAUDE.md` | Main-session orchestration instructions. |
| `.claude/agents/` | The five agent definitions. |
| `campaigns/_template/` | Blank campaign config — copy it, never edit it directly. |
| `campaigns/<your-campaign>/` | One real campaign's `campaign.yaml`. |
| `workflows/` | Daily sequence, approval flow, browser policy, manual-prospecting fallback. |
| `state/` | Cross-campaign memory: active campaigns, action/contact/signal history, permission ledger, pending approvals. |
| `outputs/` | Daily reports, approval queue, manual actions, and research/content working files. |
| `docs/` | This file, the skill inventory, the tool/connector inventory, and the automation readiness report. |
| `logs/` | Run logs and errors. |

## Onboarding a new campaign

1. Copy `campaigns/_template/` to `campaigns/<campaign-id>/`.
2. Fill in every field in its `campaign.yaml` — target accounts, personas, messaging, data sources, `crm_destination`, `approval_mode`, limits, and cooldown.
3. Set `status: active`.
4. Start with `approval_mode: drafts_only` (the template default) until you're comfortable moving to `batch_approval` or beyond — see `workflows/approval-workflow.md`.
5. Run the daily workflow (`workflows/daily-abm-workflow.md`) once manually before scheduling it.

## The safety model, in plain language

- Nothing gets sent, published, or posted, and no CRM record changes, unless the campaign owner has actually authorized it — recorded in `state/permission-ledger.yaml`.
- You approve a *batch*, a *category*, or a *campaign*, not each individual email or invite — see `workflows/approval-workflow.md` for exactly how that works.
- If a tool or connector isn't actually available, the system tells you what to do manually instead of pretending it happened — see `outputs/manual-actions/`.
- Every signal, contact, and draft carries its evidence and source. No agent invents a fact, a claim, or a contact.

## Further reading

- `docs/skill-inventory.md` — every existing Skill this system reuses, and which agent owns it.
- `docs/tool-and-connector-inventory.md` — what's actually connected right now.
- `docs/automation-readiness-report.md` — what can run unattended today vs. what needs your review, and a recommended first campaign.
