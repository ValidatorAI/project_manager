---
name: bot-approvals
description: Use when a bot action needs human approval before it runs.
version: 0.1.0
author: siavash (drafted by Hermes)
license: MIT
metadata:
  hermes:
    tags: [pwr, bots, approvals]
    related_skills: [pwr-entities, bot-messaging, task-management]
---

# Bot approvals — the requester's seat

Purpose: when and how a bot raises an approval request and behaves around its outcome. Distinct on purpose: workspace's action skills cover raising AND resolving/consuming approval events across the company; you are only ever the **requester**. Verified tools (workspace soul's approvals row): `add_approve_request_with_message`, `add_approval_request` + `add_message` (split form), `approval_requests`, `get_approval_request`, `edit_approval_request`, `delete_approval_request`. Statuses: `pending` → `approved` / `confirmed` / `denied` / `canceled`.

## Gate: does this action need approval?

Needs approval (user-visible, high-impact): company-home items, company/project status updates, milestones / todos / bottlenecks, decision records & ADRs, all-hands records, knowledge proposals, room-level decisions, releases/deployments, any external send (message to a room you don't own the thread of, third-party systems).

Does NOT need approval: work inside the explicit scope of a direct user request ("fix X", "add note Y" said by the user), private drafts, reads, logging writes.

When unsure, prefer one visible question via `add_action_message` over a silent guess or an over-cautious approval that trains users to rubber-stamp.

### Task-inbound ("Gated action:" in your AiTask description)

Message_analyzer routes gated chat work to you with a description beginning `Gated action:` and citing a source message id.

**Pre-flight (AP-11): is it REALLY gated?** Re-check the action itself against the gate section before raising — even when the description says `Gated action:`. A non-destructive, explicitly user-scoped instruction (a rename, title/wording fix, edit of a private draft, a note) needs NO approval: do the work and log it; if the target artifact does not exist, post ONE room message reporting the missing target and offer to register the intent as a todo — do NOT raise an approval for it. Wrong: approval #22 raised to rename a not-yet-filed FAQ draft title. Right: "No FAQ draft is filed yet — nothing to rename. Want me to note it as a todo for when it lands?" Only keep the gate when the action is destructive/irreversible, spends money, sends externally, or mutates shared state.

**Direct action:** descriptions beginning `Direct action:` are pre-cleared: execute (or report the missing target per above), log via `create_tool_cal_log`, close with cost — never raise an approval for them.

When you receive a genuinely gated one:
1. Your FIRST room-visible step is raising the approval: `add_approve_request_with_message` in the cited room, `body` = the exact action plus the 4-tuple (target+ids, scope, boundary, evidence), payload decision-text mirroring the requested outcome.
2. Record the approval id in `actual_task_context`, state in the room/task = **waiting-for-approval**, task stays open (never dispatched-closed).
3. Act only on an `approved` decision event for THAT approval id; cite it in your completion log. `denied` → record the reason, close the task per close-out, do not re-raise the same scope.
4. After the approved decision arrives: execute within the approved scope, then — BEFORE closing the task — produce the evidence the approval promised: the artifact/receipt for money or external sends (plus its `create_tool_cal_log` row), a completion message in the cited room citing approval id `<id>` (AP-14), and only then `ai_task_cost` → `update_ai_task(finished: true, actual_task_context: ...)`. A task closed without its promised evidence is a failed approval.


## Raising (the one-call rule)

Post the intended action and open the approval in the SAME call:

`add_approve_request_with_message(room_id | room_name, body, request_type, payload{title, decision-text})`

- `body` = the exact action you will perform — a human should be able to approve it without imagining the rest.
- A creator identity is required at runtime though the schema does not mark it: pass `user_name` (or `bot_name`) = your own profile name and the one-call raise SUCCEEDS. Verified 2026-11-16: `add_approve_request_with_message(room_id 6, user_name "project_manager", body, request_type, payload)` created msg 648 + approval 25 pending first try. Omitting the creator aborts 'Validation failed: Creator must exist'. Both abort shapes write ZERO rows, so correct the arguments and resend — never assume a half-posted message, and never re-raise when a prior attempt returned an error. Older 'platform limitation, use split form' precedents (approvals 16/23) are superseded.
- Payload SHAPE can abort the whole raise with an opaque bridge error: a raise whose `payload` carried nested array/object VALUES got 'MCP server pwr-action rejected the last 4 calls... Paused for ~25s' with no error text and ZERO rows written (2026-11-16, approval 27); the identical content resent with FLAT string payload values (lists folded into prose) succeeded first try after the pause. Recovery: wait out the pause, never double-raise, flatten payload values to strings and retry exactly once.
- The approval must carry: the **target** (entity + ids), the **scope** (project/room, budget if any), the **boundary** (what you will NOT touch), and the **evidence** you will produce on execution.
- Cost/money approvals must state the UNIT explicitly and disambiguate a bare figure by matching it against the target entity's budget fields (a quoted 'vs 10000' equal to `usd_budget 10000.0` means USD via `usd_used`, not tokens); write that interpretation into the payload and offer correction in the decision text before approval, since the wrong unit on an approved write is silently executed.
- The split form (`add_message` first, then `add_approval_request` carrying `message_id`) is the proven fallback when the one-call raise still aborts 'Creator must exist' after the corrected resend: `add_message` resolves the creator fine when given `user_name`. Record the substitution and its reason inside the payload (`request_type_note`) so the gate stays explainable. Use the split form also when the message predates the approval decision.
- Last resort when the seat has no usable message creator in any candidate room (one-call raise aborted AND split `add_message` Forbidden — test both rooms before concluding): a direct approval row with NO message via `add_approval_request(room_id, request_type, payload)` omitting `message_id` — the gate row still creates and stays pending. The payload must then carry the entire report a body would have held (title, decision, evidence, dedupe, boundary, on_execute steps) plus a `channel_note` naming each failed attempt, and the failed channel gets its own `create_tool_cal_log` audit row per the failed-channel precedent. Only degrade here after the creator-corrected resend of the one-call has itself failed — 'Creator must exist' most often means the omitted `user_name`, not a dead seat.
- People named in the source material (decision owner, approver) with no `get_users` row are gates, not assignees: state them as unresolved in the body, list them in `payload.open_questions`, and name who acts as approver meanwhile — never assign work to a phantom.
- Coverage gaps the plan crosses belong in the SAME gated payload as explicit adds: before planning with dates, enumerate `company_status_period` (company-wide and sparse); a project ending after the active period's `ends_on` requires creating the next period as part of the approved change set, not as a later unilateral write.
- Dedupe first: list `approval_requests` for an existing pending request on the same target — never open a second gate for one action. The same sweep doubles as the STYLE template: derive `request_type` naming (descriptive snake_case) and payload keys (title, decision, gating_inputs, not_touched, on_execution, on_denial, dedupe, verified_state, refs) from live rows, never invented.
- Verify at raise-time, not from the routing sweep: a task description's stale findings are not facts — tracker rows it calls missing may exist by your pass (rows can land in the same minute as the source message), and an approval id it cites may already be resolved. A status-filtered sweep structurally hides resolved rows, so fetch every cited id by `request_id` for its live status before referencing it in the body; write `verified_state` only from these fresh reads, and correct stale topic memory in the same pass.
- A room is a RUNTIME requirement even though the schema marks only `body` required: a raise aborts with validation error 'Room not found' and writes ZERO rows (no message, no approval) when (a) it carries `project_id` but no `room_id`/`room_name`, or (b) it carries BOTH `room_id` and `project_id` while the room's own `project_id` is NULL — the room lookup is scoped to the project and an unlinked room fails to resolve. So once a room is verified, raise with `room_id` ALONE. Do not rescue a roomless project by borrowing another project's room; an unlinked/private room that happens to appear in `list_rooms` is not automatically wrong either — it must be VERIFIED before posting: a workspace-scaffolded project's room typically arrives with null `name` and null `project_id`, and is confirmed as THIS project's room when its `created_at` matches the project's creation to the second and `get_room_members` lists a plausible approver. When no room verifies, the gate cannot be raised: record 'missing room' as the exact unblock input on the task (`description` + `actual_task_context`, stay dispatched-open) and ask the operator — room provisioning is workspace's specialty, route a child AiTask for it only when directed.
- A room that EXISTS but carries `name: null` and `project_id: null` (an auto-created project companion) is unresolvable by the approval tools: both `add_approve_request_with_message` and `add_approval_request` abort with 'Room not found' even with a correct `room_id` — int, string, or attached via `message_id`, all fail — while the message tools (`add_loading_message`, `add_message`) post into the same room fine. Recovery, gate not skipped: post the full proposal via `add_message` (body carries the exact change + payload fields and names the fallback in-text); log every failed attempt and the resolver reason in one `create_tool_cal_log` row citing the posted message id; hold the task dispatched-open with the missing-approval-row failure stated in `actual_task_context`; consent recorded by the human in operator chat then stands as the approval evidence. Ask the room owner to name/link the room — a null-name room cannot host a formal gate; never borrow another project's room meanwhile.

## While waiting

- Your state is **waiting-for-approval** — never "done", never "executed". Say exactly that in the room/task context.
- Keep the task open; record the pending approval id in `actual_task_context`. Do not pre-execute any part of the approved action "just in case".
- The raise's own room message satisfies the surface-it duty for that gate: on any later pass over the same still-pending approval, add no new room message and no second surfacing (that double-pings the approver) — the observable no-op cal-log is the entire response. Re-surface only if the raise never produced a room footprint.

## Outcome handling

The decision arrives as `approval_request_approved | confirmed | denied | canceled` (delivered through the workspace event flow; you'll learn of it via your task context, room message, or a follow-up task — you do not consume the raw event queue).

- approved / confirmed → execute exactly what was approved; then log every action with `create_tool_cal_log` and note the approval id you acted under.
- denied / canceled → **do not proceed at all**. Record the denial in the originating task's `actual_task_context`, dismiss any related attention item you created (`edit_<category>` with `status: "dismissed"`), close the task per task-management rules. A denial is a legitimate, complete outcome.

## Hard don'ts

- Never `add_decision_message` on your own request — that channel carries the human's decision.
- Never approve, confirm, or deny on a human's behalf.
- Never resubmit a denied action unchanged; if you believe the denial was a misunderstanding, raise a NEW request with the misunderstanding addressed in the body, or ask via `add_action_message`.
- Never let an approval lapse silently: if a pending request blocks your task, surface it (loading-message update or `add_action_message`) instead of idling.

## Worked example

```
User asks coder in Engineering room: "publish the new onboarding doc to company knowledge"
→ publishing to project knowledge is user-visible → needs approval
→ approval_requests(project 3): no pending match
→ add_approve_request_with_message(
     room_name "Engineering", request_type "knowledge_publish",
     body "Publish <doc> to /company/3/projects/3/knowledge as note 62 draft;
           will NOT touch ADRs or directory; evidence: item id + path after write",
     payload {title: "Publish onboarding doc", decision: "Approve publishing..."} )
→ room update: "Waiting for approval (request 41) — will publish on approval."
→ event: approval_request_confirmed → write the note → create_tool_cal_log(
     used_tool_name add_project_note, target_entity_id 62, change_log "published onboarding note")
→ final add_message with the id + path.
```
