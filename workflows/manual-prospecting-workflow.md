# Manual Prospecting Workflow

What happens when a needed source, platform, or action has no automated path — no connector, no reliable browser path, or it's explicitly blocked by Door-Keeper. The rule is simple: **never fabricate, never silently skip — hand the human a specific, actionable request instead.**

## When this workflow triggers

- `docs/tool-and-connector-inventory.md` shows the needed capability as "Requires user permission," "Not connected," or "Manual process required."
- `door-keeper` returns `BLOCKED` for an action that would otherwise be needed.
- A source (e.g. TeamBlind, a paywalled report, a login the user hasn't authorized Claude in Chrome for) can't be reached reliably by any connected tool.
- `account-buying-committee` or `account-signal-intelligence` cannot find a required contact or signal through any available source.

## What to write

Create one file per manual action in `outputs/manual-actions/`, named `<campaign_id>-<short-description>-<date>.md`, containing:

```yaml
campaign_id:
requested_by_agent:
date:
what_is_needed:
why_it_matters:
suggested_source_or_platform:
suggested_search_or_action:
what_to_return_to_claude:      # exact format so it can be fed back in cleanly
blocking_reason:               # not connected / requires permission / captcha / login wall / rate limit / no reliable source
priority:
```

## Using `manual-lead-capture` as the fallback skill

When the gap is specifically "I found a source but have no connector for it," delegate to the `manual-lead-capture` skill rather than improvising: it asks for source type, source location, destination format, required fields, and whether email is required, and it never scrapes, exports, downloads, or bypasses access controls. It only ever uses information already visible or provided.

## Feeding results back in

Once the human completes a manual action (runs a search, copies information, authorizes a connector), the returned information goes through the same pipeline as anything else — `data-cleaner` before it's treated as a contact record, `fact-checker` before a claim from it is used publicly. A manually-sourced record is not exempt from suppression, cooldown, or evidence requirements.

## Reporting

Every open and resolved manual action for the period shows up in the daily report's "Manual actions" section (see `workflows/daily-abm-workflow.md`) — this workflow exists so gaps are visible and actionable, not so they disappear.
