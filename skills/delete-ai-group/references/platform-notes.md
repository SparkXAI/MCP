# Platform notes - managed-group write tools

Shared behavior for the managed-group write tools (`create_sd_ai_managed_group`,
`save_sp_sb_ai_managed_group`, `edit_sd_ai_managed_group`, `delete_ai_managed_group`).
This file ships inside each managed-group skill so it travels with the skill.

## Enforce the constraints yourself - for clear errors and to catch what slips through

When a user acts through the platform UI, the front end blocks invalid input (greys
out fields, hides unsupported options, caps ranges). **Through MCP, none of that UI
guarding runs.** Prod testing (2026-08-13) shows the backend now validates a lot on its
own - enum/type checks, ACOS/ROAS/personality ranges, `*Type`<->value companions, `ids`
size, profile authorization, `remark` length - and **rejects** those bad calls rather
than silently accepting them. That's good, but it isn't a licence to send sloppy input:
some combinations are still accepted unexpectedly or silently ignored (e.g. an
unsupported ad-type action-space switch on the wrong ad type), and a downstream
rejection is a worse experience than a precise up-front message.

Therefore: **treat the constraints in this skill as rules you enforce yourself, before
sending** - both to give the user a clear error and to catch the cases the backend
doesn't.

## Unsupported features -> tell the user, don't send

Some settings don't apply in a given context - a wrong ad type, an unsupported mode, or
a feature not available yet. Per the 2026-08-14 backend spec these should be **rejected
with an error**, but where that validation isn't implemented yet the call may instead
return `success` with **nothing happening** (silently ignored). **If the user asks for
something not supported in their context, tell them plainly and skip that field - never
send it silently and never present it as if it took effect.**

## Auth & scope

- Authenticate via bearer token. Always call `get_user_authorized_context` first to
  get authorized `profileId`s; pass only an authorized one.
- `tenantId` / `userId` / `userName` are injected server-side from the token - never
  pass or fabricate them.
- Writes require the managed-group write scope. **Delete needs its own separate
  scope** - a token that can create/edit may still not be able to delete.

## Response envelope

Success:
```json
{ "isError": false, "toolName": "<tool>", "data": { "status": "success", "result": { "aiGroupId": 827136 } } }
```
Failure:
```json
{ "isError": true, "data": { "error": "<message>", "recoveryHint": "<hint, if any>" } }
```

Always check `isError` before reading `data`. When `recoveryHint` is present, use it and
relay it to the user.

**Batch results are three-state - don't judge by `isError` alone.** A batch operation
can partially succeed and still come back with `isError:false`:

```json
{ "isError": false, "data": {
    "status": "partial_failure",
    "result": { "success": 1, "fail": 2 },
    "failedItems": [ { "aiGroupId": 1002, "error": "aiStatus is not on" } ] } }
```

`status` is `success` / `partial_failure` / (all-failed ->) `isError:true`. If
`failedItems` names the failures, derive and verify the succeeded set. If it is empty,
re-read every requested id because counts alone cannot identify which items succeeded.
Never present a partial result as full success.

## Errors: prefer the hint, but don't rely on it

The failure shape carries `data.error` and an optional `data.recoveryHint`, and the
batch layer translates some downstream codes into readable messages (e.g. "Budget must
be between 50 and 500", "Operation 'budget' requires: budgetType"). But **not every
error is populated with a useful hint** - some downstream business errors still come
back generic (e.g. `"业务执行错误"`, `"Resource Not Found"`), and parameter values are
validated downstream, not by the tool. So: get the request right the first time by
sticking to the documented fields/enums; relay `recoveryHint` when it's there, and don't
assume it always will be.

## Non-idempotent + 30s timeout

Create and SP/SB save are **non-idempotent** - each call creates/changes state. The
downstream call can take up to ~30s; on timeout the operation may already have
applied. **Always verify with a `get_entity_metadata(profileIds=[...],
entity='aiGroup', userContext='...')` read before retrying** - a blind retry can
create a duplicate group. (`get_entity_metadata` requires `profileIds`, `entity`,
`userContext` on every call.)

## Verify, don't trust the envelope

A `status: success` envelope does not by itself prove the intended state. Some edits
on an AI-running group can be silently skipped downstream. After any write, read the
group back with `get_entity_metadata(profileIds=[...], entity='aiGroup',
userContext='...')` and confirm the fields you set actually took the values you
expected.

For multiple writes against the same object set, execute serially and read back after
each call. Never run dependent writes concurrently. When operation-log read access is
available, also verify the resulting audit entry; `changedBy` is server-derived.
