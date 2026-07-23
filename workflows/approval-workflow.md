# Approval Workflow

This system defaults to draft-only. Nothing external happens without an authorization recorded in `state/permission-ledger.yaml`. This file defines the approval modes, the permission ledger, and how a batch request avoids asking the campaign owner the same question once per contact.

## Approval modes

Set per campaign in `campaign.yaml`'s `approval_mode`:

| Mode | What agents may do without asking again |
|---|---|
| `research_only` | Research, analyze, score, recommend. No drafting, no external actions. |
| `drafts_only` | Everything in `research_only`, plus create drafts (email, comment, LinkedIn note). Nothing external. **This is the default for a new campaign.** |
| `batch_approval` | Everything in `drafts_only`. External actions execute only after the owner approves a specific consolidated batch request. |
| `campaign_preapproved` | External actions in the campaign's `preapproved_action_categories` execute automatically, within daily limits and cooldowns, for the life of that authorization. Anything outside those categories still needs a batch request. |
| `approved_automation` | The broadest mode. Defined action categories, platforms, and volumes run unattended within recorded limits. Still never overrides Door-Keeper safety checks (suppression, cooldown, opt-out, platform warnings), and still never bypasses a browser/connector's own unavoidable confirmation prompts. |

Escalating from one mode to the next is always an explicit owner decision recorded in the campaign's `campaign.yaml` — agents never raise their own approval mode.

## The permission ledger

`state/permission-ledger.yaml` holds every authorization. Each record:

```yaml
authorization_id:
campaign_id:
authorized_action_categories: []   # e.g. ["linkedin_invite", "email_send"]
authorized_platforms: []
authorized_scope:                  # which accounts/contacts this covers
  accounts: []
  contacts: []
max_volume:
approval_mode:
approved_by:
approved_at:
expires_at:
status:                            # active | expired | revoked
restrictions: []
revoked: false
```

## How a batch request gets created

1. `outreach-engagement-followup` (or the Director) collects every proposed external action that isn't already covered by an active authorization.
2. It writes **one** consolidated request to `outputs/approval-queue/`, containing: platforms involved, number of actions, accounts and contacts involved, action types, applicable daily limits, representative example content (not every single draft), and any risks or exclusions worth flagging.
3. The owner reviews and approves the batch, the campaign, or a specific action category — not each item individually.
4. Once approved, a new record is appended to `state/permission-ledger.yaml` with its scope, volume cap, and expiration.
5. From then on, matching actions execute (or continue queuing for execution) without asking again, until one of the re-authorization triggers below fires.

## When to ask again

Request new permission only when:

- The existing authorization expired.
- The campaign changed.
- The platform changed.
- The action type changed.
- The approved volume would be exceeded.
- The recipient/account scope changed materially.
- The content changed materially from what was approved.
- A safety concern was detected (opt-out, complaint, platform warning, suppression hit).
- Claude in Chrome or a connector requires an unavoidable system-level confirmation — surface that prompt as-is.

## What never gets automated, at any approval mode

- Stopping outreach after a reply, opt-out, complaint, or negative response is a mandatory safety action, not something that needs approval — it happens immediately regardless of mode.
- No local permission record ever overrides Claude's, Chrome's, a connector's, an organization's, or a website's own security controls.
- Agents never modify the user's browser permission settings.

## Where things live

- Pending items the owner hasn't decided on yet: `outputs/approval-queue/`.
- What's currently authorized: `state/permission-ledger.yaml`.
- What's waiting on a decision across all campaigns, mirrored for quick scanning: `state/pending-approvals.yaml`.
