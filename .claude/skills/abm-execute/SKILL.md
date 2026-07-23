---
name: abm-execute
description: Execute a single reviewed-and-authorized ABM batch — sync approved contacts to Monday ABM Prospects, enroll them in exactly the selected email platform, and send only approved LinkedIn actions. Manual-invoke only; never runs on its own. Use ONLY when the user explicitly types /abm-execute [batch-id-or-date].
argument-hint: "[batch-id-or-date, e.g. today or 2026-07-23]"
arguments:
  - batch
user-invocable: true
disable-model-invocation: true
effort: high
---

# /abm-execute

Executes **one** reviewed, owner-approved batch. This is the only skill in the system that performs external writes. It is **manually invoked by the owner only** (`disable-model-invocation: true`) — Claude must never trigger it on its own, never as a follow-on to `/abm-daily`, and never to "finish the job."

`$batch` identifies the batch: `today`, a date (`2026-07-23`), or an explicit batch/request id.

## Preconditions — refuse to proceed unless ALL hold

1. **Active Door-Keeper authorization** exists in `state/permission-ledger.yaml` for this campaign, covering the exact action categories in the batch, unexpired, not revoked. The standing *research* authorization does **not** count — execution needs a reviewed *execution* authorization. If none, stop and tell the owner what to approve.
2. The batch exists and its contacts are marked **Approved for Outreach** in the Google Drive Prospect Review Sheet.
3. A concrete email platform is selected (`apollo` or `mixmax`) — never both for the same contact.
4. **A sender alias has been explicitly selected by the owner for THIS execution and live-verified** (see "Sender alias selection" below). If no alias has been selected this run, STOP and ask before doing anything else — never assume, never reuse yesterday's alias, never default to the Apollo account owner's mailbox.

## Sender alias selection (MANDATORY before any enrollment, activation, scheduling, or send)

The sender mailbox/alias is **never** assumed permanently. Before every `/abm-execute` run, in this order:

1. Perform a **live, read-only** check of the selected email platform (Apollo: `apollo_email_accounts_index`; Mixmax: the connected-mailbox read) to retrieve the **currently connected sending mailboxes/aliases**.
2. Confirm each alias is **authorized and available** (connected, not disconnected/restricted).
3. Ask the owner exactly: **"Which sending alias should be used for this execution?"**
4. **Show the available aliases** (from the live check) and **show the campaign's `preferred_sender_alias`** (may be listed first, but is only a suggestion — never auto-selected).
5. **Wait for the owner's answer.** Do not start enrollment, activation, scheduling, or sending until an alias is explicitly selected. The owner may reply with just the email address; interpret that as the alias for **this approved batch only** (not a permanent default unless they say "Save this as the preferred alias for this campaign," which routes to `/abm-campaign`).
6. **Record the selected alias in the Door-Keeper execution authorization** (`state/permission-ledger.yaml`) with `alias_selected_by`, `alias_verification_status`, and `alias_verification_time`.
7. Use **only** that alias for the current execution. **Do not silently rotate** to another mailbox. Mailbox rotation is disabled unless the owner explicitly approves rotation for this execution.
8. **If the selected alias is unavailable, disconnected, restricted, or missing:** STOP, show the exact issue, and ask the owner to select another available alias — never substitute one automatically.

The sequence itself may be created before the alias is selected (platform permitting), but enrollment/activation/scheduling/sending must not.

## Steps

1. Verify the active execution authorization (above).
2. **Run the Sender alias selection flow (above) and record the owner-selected, live-verified alias.** No enrollment/activation/scheduling/sending until this is done.
3. Read only the contacts marked **Approved for Outreach** for `$batch` from the Drive review sheet.
4. **Re-check live** against Monday, Apollo, Mixmax, suppression, cooldown, and sales ownership — a contact that changed status since review is dropped from the batch with a reason.
4. Sync approved contacts to **Monday ABM Campaign Prospects** (set Monday Sync Status = Synced). **Never** add unreviewed contacts to Monday.
5. **Never** add contacts directly to the sales team's active boards — ABM prospects only.
6. Enroll approved contacts in **only the selected email platform**, sending **only from the owner-selected, live-verified alias** (Apollo: set it during enrollment/scheduling; Mixmax: select the approved connected mailbox before activating). **Never** enroll the same person in both Apollo and Mixmax, and **never** send from an alias the owner did not select this run.
7. Map the approved personalized fields (Apollo field mapping or Mixmax merge fields as applicable).
8. Start/activate only the approved contacts' sequence membership.
9. Send only the approved LinkedIn connection requests (≤ the approved count).
10. Publish only the approved LinkedIn comments.
11. **Connector-first**, then Claude in Chrome only when the connector cannot complete the approved action, then manual action.
12. **Stop immediately** on CAPTCHA, login prompt, rate limit, suspicious-activity/security warning, or platform restriction — surface the actual prompt, never work around it.
13. Capture confirmation evidence for every action (ids, timestamps, tool responses).
14. Update Google Drive (statuses), Monday, Apollo/Mixmax, and local state + `logs/` + the dashboard.

## Never expand beyond the approved batch

No new prospecting, no contacts outside the approved list, no channels or volumes beyond what was authorized. If the owner wants more, that's a new `/abm-daily` → approval → `/abm-execute` cycle.

## One-word `Approve` behavior

When the owner replies `Approve`, that approves the latest final execution package but **does not** remove the sender-alias requirement. If no alias has been selected this run, respond: **"Batch approved. Which sending alias should be used for this execution?"** — show the live available aliases and the preferred alias — and wait. Only after the owner selects an alias do you record it, confirm the execution scope, and return the exact `/abm-execute` command. A reply containing only an email address selects that alias for the latest approved batch only.

## Report

Completed actions · failed actions (with the tool's actual error) · blocked actions · pending actions · manual actions · messages sent (tool-confirmed only) · contacts enrolled · LinkedIn actions completed · confirmation evidence. Never mark anything `sent`/`enrolled`/`synced` without the platform confirming it. Enrolled ≠ sent; invitation sent ≠ accepted; comment prepared ≠ published.

**Also record these alias fields** in the execution log, daily CSV, Google Drive review data, dashboard, and EOD report: preferred campaign alias · available aliases · selected execution alias · alias selected by · alias verification status · alias verification time · sequence platform · emails scheduled · emails sent · failures caused by alias/mailbox issues. Never expose passwords, tokens, authentication details, or sensitive mailbox configuration.

## Delegation

Execution belongs to the `outreach-engagement-followup` agent under the Director's coordination, gated by the `door-keeper` skill. It uses `sequence-sender` (Mixmax), the Apollo connector (enrollment), `connection-helper` (LinkedIn), and `record-keeper`. It does not invoke further subagents.
