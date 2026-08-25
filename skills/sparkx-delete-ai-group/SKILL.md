---
name: sparkx-delete-ai-group
description: >-
  Delete (archive) an AI managed group (托管组) itself - SD, SP, or SB - releasing its
  campaigns or migrating them to another group. Use when the user wants to delete / 删除 /
  归档 a managed group (the group container) or wind one down. Destructive and
  irreversible. This archives the GROUP - it is NOT for removing a single campaign from a
  group (use sparkx-edit-ai-group), editing config (use sparkx-edit-ai-group), or creating a group (use
  sparkx-create-ai-group).
metadata:
  version: 1.0.3
---

# Delete AI Managed Group

Delete (archive) a managed group via `delete_ai_managed_group`. This is
**destructive and cannot be undone** (`destructiveHint=true`) - treat it with more
care than any read or edit. Read
[`references/platform-notes.md`](references/platform-notes.md) once first (auth,
response envelope, generic errors, the separate delete scope).

**Scope**: `amazon_sa_managed_group_delete:write` - separate from the create/edit scope
(`amazon_sa_managed_group:write`). A token that can create and edit groups may still be
unable to delete one; if the call fails on permissions, name this scope to the user rather
than retrying.

**Routing is fail-fast.** The tool reads the group's `campaignType` to pick its delete path
(SP/SB and SD go to different backends). If that type can't be determined or isn't one of
`sponsoredProducts` / `sponsoredBrands` / `sponsoredDisplay`, the call is **rejected** with a
fail-fast error and **nothing is archived**. Treat that error as "the group
couldn't be identified", not as "the group was partially deleted": re-read the group with
`get_entity_metadata(entity='aiGroup')` and confirm its state before doing anything else.

## What "delete" does - and the campaign disposal choice

Deleting archives the group. Its campaigns don't vanish - you must say what happens
to them via `type`:

| `type` | Behavior | When |
|---|---|---|
| `1` (release) | Unbind all campaigns from the group, then archive it | The campaigns should go back to being unmanaged |
| `2` (migrate) | Move campaigns to `targetAiGroupId`, then archive this group | The campaigns should keep being AI-managed under a different group |

`type=2` requires `targetAiGroupId` (an existing group). Always confirm the disposal
with the user - "release" and "migrate" have very different consequences for their
live campaigns.

## Parameters

| Field | Type | Required | Notes |
|---|---|---|---|
| `profileId` | long | **Yes** | Shop ID from `get_user_authorized_context` |
| `aiGroupId` | long | **Yes** | The group to delete |
| `type` | int | **Yes** | `1`=release (unbind), `2`=migrate to target |
| `targetAiGroupId` | long | **type=2 only, required** | Destination group for migration; must exist |

> **Delete is one group per call - there is no `ids` / batch parameter** (confirmed
> against the current tool schema, incl. the forward-looking version). `aiGroupId` is a
> single id. To wind down several groups, loop over single deletes **serially** - see
> [Deleting more than one group](#deleting-more-than-one-group). Never assume a batch
> delete exists or construct an `ids` array.
>
> **`type` has no safe default.** `type=2` without `targetAiGroupId` is **rejected** by
> the backend - do not silently fall back to `type=1` (release) and do not guess a
> migration target. If the user hasn't said release-vs-migrate, ask.

## Workflow

1. **Resolve the group to an ID.** Users refer to groups by name - look it up with
   `get_entity_metadata(profileIds=[...], entity='aiGroup',
   filters={"aiGroupName": {"like": "%<name>%"}}, userContext='...')` (every `get_entity_metadata`
   call requires `profileIds`, `entity`, `userContext`). The `aiGroupName` filter is a
   substring `like` - exact-match the name in the returned rows. Get its `aiGroupId`,
   and read `aiStatus`, `numCampaign`, `campaignType` for the checks below. If more than
   one group matches, list the candidates and let the user pick - never guess which one
   to delete.
   - **Also capture the group's full campaign list.** Query `entity='campaign'` **filtered
     by this `aiGroupId`** (server-side filterable for campaigns), **fully paginate**, and
     **save the campaign IDs** - you need the actual IDs (not just `numCampaign`) to verify
     the disposal after deleting.
2. **Pre-flight checks:**
   - **AI must not be running.** A running group can't be deleted. `aiStatus`:
     `1`=running, `2`=turned off, `0`=never enabled - only `1` blocks deletion. If
     it's running, see "When AI is on" below.
   - **`type=2` (migrate) - validate the target, not just its existence:**
     - `targetAiGroupId` exists (another full-signature `entity='aiGroup'` lookup);
     - it belongs to the **same `profileId`**;
     - it is **not the source group itself**;
     - its `campaignType` is **compatible** with the campaigns being moved.
     If target capacity or another acceptance rule is not exposed by the read tool,
     do not claim it was pre-validated; let the backend gate it and verify every
     campaign after the call.
     Also make the user aware the migrated campaigns will **adopt the target group's
     optimization config**, not keep the source group's.
3. **Confirm before deleting.** Echo back exactly what will happen - the group's name
   and id, and the disposal ("release N campaigns" or "migrate N campaigns to group
   X") - and get an explicit yes. This is irreversible; a vague "yes go ahead"
   earlier in the conversation is not enough.
4. **Execute** `delete_ai_managed_group`.
5. **Verify - the archive AND the campaign disposal.** Don't report success from the
   tool envelope alone:
   - Re-read `entity='aiGroup'` -> confirm the group is gone/archived.
   - **`type=1` (release):** re-read the **saved campaign IDs** (`entity='campaign'`)
     and confirm each one's `aiGroupId` is now cleared.
   - **`type=2` (migrate):** re-read the **saved campaign IDs** and confirm each one's
     `aiGroupId` now equals `targetAiGroupId` - checking only the target's campaign
     *count* doesn't prove the *right* campaigns moved.
   - If operation-log read access is available, query `get_operation_log` for the group
     and deletion window and confirm the release/migration and identifiable operator
     were recorded. `changedBy` comes from the authenticated token; never fabricate it.
     Without log access, report that object state was verified but audit logging was not.

## Deleting more than one group

There is no batch delete - do them **one at a time, serially**, never concurrently:

1. **Resolve all target groups up front** and read each one's `aiGroupId`, `aiStatus`,
   `numCampaign`, `campaignType`, and full campaign-id list (per the workflow above).
2. **Screen for running AI first.** Any group with `aiStatus=1` can't be deleted (backend
   returns `aigroup AI is running`). Don't delete some and then stall on a running one -
   surface the running groups first and let the user decide (turn AI off themselves, or
   drop those groups) before you begin the loop.
3. **For `type=2` (migrate), validate the target against *every* source group** - same
   `profileId`, target != any source, and `campaignType` compatible with all groups being
   moved. A target that fits one group but not another can't batch them together.
4. **Confirm the whole set once, explicitly** - list every group (name + id) and its
   disposal - then delete **each** with its own `delete_ai_managed_group` call, reading
   back that group's campaign disposal before moving to the next.
5. **Verify per group, not by count.** Re-read each group's saved campaign ids and confirm
   the release/migration actually landed. A partial run leaves some groups deleted and
   others not - report exactly which.

## When AI is on (aiStatus=1)

A running group must have AI turned off before it can be deleted. **Do not turn AI
off on your own** - turning off AI stops live optimization and is a real change the
user should decide on. Instead:

1. Tell the user the group's AI is running, so it can't be deleted yet.
2. Offer to turn it off first, and get an explicit yes for *that* step specifically.
3. Only after they confirm, turn it off (via `sparkx-edit-ai-group` / setting AI status to
   off), then proceed to the delete (which is its own confirmation).

If the user declines, stop - don't delete, don't silently flip AI off.

## Response & errors

Success: `{ "isError": false, "data": { "status": "success", ... } }` -> then verify
with a read.

Errors come back as `{ "isError": true, "data": { "error": "...", "recoveryHint":
"..." } }` - relay `recoveryHint` when present. (Some failures instead use the top-level
`errorType`/`error` shape documented in `references/platform-notes.md`; check `isError`
first, then look for either form.) It isn't always populated (some come
back generic, e.g. `"Resource Not Found"`, `"aigroup AI is running"`), so map them:

| Symptom | Likely cause | What to do |
|---|---|---|
| "aigroup AI is running" | AI still on | see "When AI is on" |
| "aigroup is not exist" | wrong `profileId`/`aiGroupId` | re-resolve the group by name |
| "param is empty" | missing required field | check `profileId`/`aiGroupId`/`type` (and `targetAiGroupId` for type=2) |
| "Resource Not Found" | group/profile mismatch, stale ID, or downstream lookup failure | re-resolve the group under the authorized profile; verify current state before deciding whether to retry |

**Don't infer resource existence from which error came back.** Delete runs the
permission check before parameter validation (a past information-leak bug where param
validation ran first is fixed), so a `not exist` vs `not authorized` vs `Resource Not
Found` difference does **not** tell you whether the group really exists under someone
else's scope. Treat them the same: re-resolve under the authorized profile; never report
to the user that a group "exists but you lack access" (or the reverse) based on the error
string.

## Reference files
- [`references/enum-i18n.md`](references/enum-i18n.md) - managed-group campaign type, status, goal, and action-setting labels used during preflight and confirmation
- [`references/platform-notes.md`](references/platform-notes.md) - shared write-tool behavior (auth, separate delete scope, response envelope, generic errors)
