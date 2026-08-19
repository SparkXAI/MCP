---
name: sparkx-edit-ai-group
description: >-
  Modify an EXISTING AI managed group (托管组) for Amazon Sponsored ads - change target
  ACOS/ROAS, budget, turn its AI on/off, personality, adjust SP/SB campaign membership, and
  the AI action space (bid / budget / target / structure optimization) - for one group or
  many at once (bulk). Use when the user wants to modify / adjust / 修改 / 调整 / 批量改 an
  existing group's settings or automations. SD uses edit_sd_ai_managed_group; SP/SB uses
  save_sp_sb_ai_managed_group (edit mode). Not for creating a group (use sparkx-create-ai-group)
  or deleting one (use sparkx-delete-ai-group).
metadata:
  version: 1.0.1
---

# Edit AI Managed Group

Change the configuration of one or more existing managed groups. Read
[`references/platform-notes.md`](references/platform-notes.md) once first - it covers
auth, the response envelope, generic errors, and the crucial fact that **MCP bypasses
the platform UI's validation, so you must enforce the front-end-only rules yourself.**

## Route by ad type

| Ad type | Tool | Reference |
|---|---|---|
| **SD** | `edit_sd_ai_managed_group` (takes an `ids` array - inherently batch, max 20) | [`references/edit-sd.md`](references/edit-sd.md) |
| **SP / SB** | `save_sp_sb_ai_managed_group`: `aiGroupId > 0` for one group; `operation + ids` for batch | [`references/edit-sp-sb.md`](references/edit-sp-sb.md) |

Outside batch mode, `aiGroupId = 0`/empty means create (that's sparkx-create-ai-group) and a
positive id edits. In SP/SB batch mode, `operation` selects batch behavior, `ids` are
the targets, and `aiGroupId` is ignored. Determine ad type from each group's
`campaignType` before choosing a mode.

## Editing a running group can be silently skipped

If a group's AI is on (`aiStatus=1`), the backend **may silently drop some of your
changes** - the call returns `success` but the setting didn't move. This is the single
biggest trap here.

- **Never turn AI off on your own to force an edit through.** Turning off AI stops live
  optimization; that's the user's decision, not yours.
- If an edit doesn't stick because AI is running, **tell the user** what didn't apply
  and let them decide (e.g. pause AI themselves, or accept it).
- Always **verify after** (below) - don't trust the `success` envelope.

## Disambiguate before writing (when in doubt, ASK - don't guess)

Everyday words - in English OR Chinese - map to DIFFERENT things in a managed group, and
the **exact field depends on the ad type (SD vs SP/SB) and the write path (flat/batch vs
action-space)**. So this section lists **business meanings only** - do NOT hard-code a
field name from it. Resolve the meaning first, then determine ad type + tool path, then
look up the exact field/param in [`references/field-reference.md`](references/field-reference.md)
and [`references/action-space-matrix.md`](references/action-space-matrix.md).

**If a request could map to more than one meaning, STOP and ask - don't pick one.**
Guessing wrong changes live spend.

**"budget" / "预算"** (e.g. "把预算调到 100" / "set the budget to 100") - candidate meanings:
- 托管组总预算 (group total budget)
- 托管组日预算 (group daily budget)
- 组内所有 campaign 日预算之和 (sum of member campaigns' daily budgets)
- 按表现调预算的固定值/百分比 (performance/dynamic budget *adjustment* - an automation, NOT
  the budget value itself)
- 预算重新分配 (budget reallocation across campaigns)
- 单个 campaign 的预算 (a single campaign's budget - campaign-level, not a managed-group op)

> `totalBudget` / `totalDailyBudget` are **read-side fields, not write params** - never
> map a budget change onto them. **The current docs don't define how they're calculated -
> don't infer a formula from the field name.** If the user wants that number changed,
> confirm the actual business meaning first (group budget? member campaigns' budgets?
> dynamic-budget strategy?). Whether these fields change after you edit the group budget
> must be verified by reading back **after** the write - don't pre-infer.
> Also, `budgetType`/`budgetRatio` live on the edit/batch path, so the exact field is
> decided *after* ad type + path, not here.

**"bid / adjust bid" / "调价 / 出价 / 竞价"** - candidate meanings (field differs by ad
type/path, look it up after): 按表现调竞价 · 广告位调价 (SP only) · 竞价范围 (min/max) ·
分时调价 (on SB this is Rule/RBA-only, and **MCP cannot read or edit RBA config** - if
that's the intent on an SB group, tell the user it's not available through MCP and do
**not** build write params). Ask which one.

**"off / pause / stop" / "关掉 / 关闭 / 暂停 / 停"** - three very different things:
- **关闭托管组 AI** (stop the group's AI optimization) - a status change. **Never do this
  on your own; confirm first.**
- **允许 AI 暂停广告活动 / 暂停商品** (`structPauseCampaign*` / `structPauseProduct*`) - this
  is an **action-space switch that *enables the AI* to pause**; turning it on does **not**
  immediately pause anything. SP only.
- **立即暂停某个广告活动** (pause a specific campaign right now) - campaign-level, **not a
  managed-group operation** and not settable through these tools.
> "暂停广告活动" is ambiguous between "let AI pause campaigns" (action space) and "pause
> this campaign now" (campaign-level, not here). **Never map it straight to a field and
> execute - ask which is meant.**

**"target / goal" / "目标"**: 推广目标 `targetType` (1 growth / 2 stability / 3 volume /
4 legacy) vs 目标 ACOS/ROAS. Also "ACOS 调到 25" (a value) vs "ACOS 降 20%" (a ratio) are
different inputs - confirm which.

**Clarification is NOT authorization.** Even after the user answers the ambiguity, only
produce a **change preview** (business meaning + resolved field + old -> new). Do **not**
call any write tool until the user explicitly confirms the object, the meaning, and the
new value.

## Workflow

1. **Resolve the group(s) and read current config.** Look them up with the full
   signature `get_entity_metadata(profileIds=[...], entity='aiGroup',
   filters={"aiGroupName": {"like": "%<name>%"}}, userContext='...')` (the name filter is a
   substring `like` - exact-match in the results). Read each group's `aiGroupId`,
   `campaignType` (routing), `aiStatus`, and the current values of whatever you're
   about to change - SP/SB edit **merges** onto current config, and you want a before
   value to verify against.
2. **Disambiguate into a single, concrete mapping** (see "Disambiguate before writing"
   above). Determine the object level, the business meaning, the value type, and the unit.
   **If more than one mapping is plausible, STOP and ask the user - do not build any params
   yet.**
3. **Build a partial update - only the fields you're changing** (only once the mapping is
   unique). Both tools take partial input (SP/SB fills the rest from current config; SD
   applies the non-null fields to the `ids`). Don't resend unchanged fields.
4. **Self-validate the front-end-only rules MCP bypasses** (see references):
   - **ACOS / ROAS / Budget type are mutually exclusive** - the `*Type` decides which
     value field to send; sending the wrong companion (or both) fails. See
     [`references/edit-sd.md`](references/edit-sd.md).
   - **Action-space support** - only enable a capability that supports AI for this ad
     type ([`references/action-space-matrix.md`](references/action-space-matrix.md)).
     If the user asks for one that isn't supported for their ad type / mode, tell them
     and skip it - don't send a silently-ignored field.
   - **Budget** - enforce only rules you can determine reliably (positive, JP
     integer-only, and the known multi-campaign minimum relationship). Site/account
     ranges are incomplete; do not invent or silently clamp an unknown limit. See
     [`references/budget-limits.md`](references/budget-limits.md).
   - **`aiPersonality` 1-5, and >=3 when `targetType=3` (volume/冲量)**; name not blank;
     any range min <= max; coupled switches carry their companion fields.
   - **The backend now rejects invalid values - pre-validate anyway** for a clear error
     instead of a downstream one (prod-confirmed 2026-08-13): `acos`/`roas` out of range
     (incl. `0`, negative, over-limit), `aiPersonality` outside `1`-`5`, `remark` over
     **200** chars, a `*Type` sent without its value (or with the wrong companion), `ids`
     empty or over **20**, and `tosMin > tosMax` are all rejected. Also enforce these
     value bounds yourself (backend may not catch them yet, per the 2026-08-14 spec):
     `budget > 0`, `budgetRatio <= 10000`, dynamic-budget `num > 0` (`numType=1` <= 1000,
     `numType=2` <= 100000), and action-space ranges (placement/B2B `0`-`900`, bid-range
     percentage `0.01`-`100`). Don't lean on the backend for these.
   - **Word-list settings are not supported.** Do not send branded, non-branded,
     competitor, harvest-blacklist, or negative-target-blacklist fields, even if the
     routed schema exposes them. Tell the user to configure word lists in the platform.
5. **Confirm before applying - especially for bulk and for running groups.** Show a change
   preview per group: **business meaning + resolved field + old -> new**, and how many
   groups are affected. For a running group, note that some changes may not apply while AI
   is on. Get an explicit go-ahead. **Clarification is not authorization** - a prior answer
   to an ambiguity question is not itself permission to write; you still need an explicit
   confirm of the actual change here before calling any write tool.
6. **Verify - read back and compare.** Re-read the group(s) and confirm each changed
   field actually took the new value. For bulk, check **every** group, not just one -
   partial success is possible. Report any field that didn't move (often because AI was
   running) instead of implying it did. For `campaignNameSign`, also re-read the
   affected campaigns and verify their actual names gained/lost `[AI]` according to
   `campaignNameRecoveryType`; the group setting alone is not enough - also cross-check
   the `campaignNameSign` entry in `get_operation_log` (prod shows it logged with
   `changedBy=manual`).
7. **Verify the audit trail when available.** If the token also has operation-log read
   access, query `get_operation_log` for the affected group(s) and time window and
   confirm the action, object, time, and identifiable operator. `changedBy` is derived
   server-side from the authenticated token - never send or fabricate it. If log access
   is unavailable, say the business state was verified but the audit entry was not.

## Unsupported scheduling

The current managed-group write tools expose no `scheduleType`, `scheduleDate`,
`scheduleStartDate`, or `scheduleEndDate` fields, and the backend **rejects** schedule
params on v1 groups (prod-confirmed 2026-08-13). Do not invent these parameters.
Creating, editing, or deleting a managed-group schedule is not available through this
MCP version; tell the user to use the platform until the tool schema adds scheduling.

## RBA mode restriction

- The tool may switch an existing group from **RBA (Rule mode) to AI mode**.
- It must **not** switch a group from **AI mode to RBA**.
- RBA condition/action details cannot currently be read or edited through MCP. Do not
  infer the current RBA configuration or attempt to preserve, modify, or recreate it.
  If the user requests an RBA configuration change, tell them to use the platform.

## Bulk / batch edits

Bulk uses an **operation-based** protocol (`operation` + `ids` + operation-specific
parameters) - **not** a bare array of ids. Read
[`references/batch.md`](references/batch.md) before any bulk edit; the essentials:

> **Server-version guard.** This batch protocol is newer than some deployed MCP
> servers. Check the routed tool separately: SD is upgraded when
> `edit_sd_ai_managed_group` exposes `operation`; SP/SB is upgraded when
> `save_sp_sb_ai_managed_group.request` exposes `operation`, `ids`, and `batchParams`.
> If the relevant check fails, do not send new-protocol parameters; tell the user batch
> is unavailable on that server and edit groups one at a time instead.

- **`operation` + `ids` + params.** One `operation` per request. SD parameters are
  top-level. SP/SB Flat parameters go in `batchParams`; SP/SB action-space parameters
  go in `aiActionSettings`/`aiAutomation`. Don't mix unrelated fields.
- **`operation` values are UPPER_SNAKE_CASE** (`STATUS`, `BUDGET`, `BUDGET_REDISTRIBUTE`,
  ...) - prod-confirmed 2026-08-13; verify the exact enum in the tool schema. For a
  single SD flat change (e.g. AI on/off), **use the batch form `{operation:"STATUS",
  status:1}` rather than a Legacy full edit** - omitting `operation` took the Legacy path
  and returned a misleading `only supports SD groups` error in pre. See
  [`references/batch.md`](references/batch.md).
- **Validate ids vs profile.** The backend enforces profile authorization and **rejects
  cross-profile batches** (prod-confirmed 2026-08-13: a cross-profile edit is denied with
  `profileId ... is not authorized for the current user`). Still read each group first
  and confirm they're all under this profile and in the token's scope, and keep every
  batch single-profile - don't send one you expect to be rejected.
- **`ids` max is 20 - for every tool.** Both `edit_sd_ai_managed_group` and
  `save_sp_sb_ai_managed_group` reject more than 20 ids per call (prod-confirmed). Over
  20 -> split into batches of at most 20 ids, run them **serially**, and read back after
  each.
- **Group by tool, and split by ad type first.** SD -> `edit_sd_ai_managed_group` (flat,
  top-level); SP/SB -> `save_sp_sb_ai_managed_group` (`request`-wrapped) - different
  shapes, see batch.md. **SP + SB can share one SP/SB batch**, but only for operations
  **both** support; **SP-only operations** (`structOptimization`, `targetPausedAdd`) must
  exclude SB ids. **SD is always its own call** - never mixed with SP/SB. The backend
  rejects a wrong-tool/wrong-type call (e.g. `campaignType must be sponsoredProducts or
  sponsoredBrands`), but split by each group's `campaignType` from metadata **before**
  sending rather than relying on that error.
- **Result is three-state.** A batch can **partially** succeed with `isError:false`.
  When `failedItems` is populated, derive the succeeded set and verify it. When it is
  empty, re-read every requested id to identify what applied. Never infer specific ids
  from counts or call a partial success a full success.
- **Mixed current state** (some on, some off): the AI-running silent-skip caveat applies
  per group - verify each.

When one user request requires multiple operations, execute them **serially** for the
same groups: call one operation, read back its result, then build the next request from
fresh state. Do not run writes concurrently. After a timeout, verify before retrying.

## Name marking

`campaignNameSign` toggles the `[AI]` label on campaign names. When turning it off,
`campaignNameRecoveryType` decides the name: `1` = keep the current name, `2` = restore
the original (pre-`[AI]`) name. Confirm which one the user wants - they're different
outcomes.

## Response & errors

Success: `{ "isError": false, "data": { "status": "success", ... } }` - a `success`
envelope does **not** prove the change applied (verify), and for bulk it can be
`partial_failure` with `isError:false` (see [`references/batch.md`](references/batch.md)).
Failures are `{ "isError": true, "data": { "error", "recoveryHint" } }` - relay
`recoveryHint` when present, but it isn't always populated, so validate up front.
**Don't confuse the read enum with the write enum, and don't assume one global write
enum.** The **read-back / query** value is `0`=never enabled, `1`=running, `2`=off. The
**write** enum depends on the tool and call path - SP/SB single-group edit (`aiStatus`),
SP/SB batch (`batchParams.status`, which uses `2`=paused), and SD batch (`STATUS`
operation, seen to use `1`/`2`) each take their own values per the routed schema and
[`references/batch.md`](references/batch.md). **Never reuse the query-returned `aiStatus`
value as a write value** - look up the correct write value for the exact path you're on.

## Reference files
- [`references/edit-sd.md`](references/edit-sd.md) - `edit_sd_ai_managed_group`: fields, ACOS/ROAS/Budget `*Type` mutual exclusion, batch `ids`, name recovery
- [`references/edit-sp-sb.md`](references/edit-sp-sb.md) - `save_sp_sb_ai_managed_group` edit mode: merge semantics, action space, campaign membership
- [`references/batch.md`](references/batch.md) - SD top-level and SP/SB wrapped batch protocols, operation parameters, ID-profile validation, three-state result
- [`references/field-reference.md`](references/field-reference.md) - exact write field names + enums (incl. edit-only `*Type` fields)
- [`references/action-space-matrix.md`](references/action-space-matrix.md) - capability support (AI / Rule / none) per SP / SB / SD, and `noRule` capabilities
- [`references/budget-limits.md`](references/budget-limits.md) - site/account-type budget ranges (front-end rules MCP bypasses)
- [`references/enum-i18n.md`](references/enum-i18n.md) - 中文 <-> English <-> code
- [`references/platform-notes.md`](references/platform-notes.md) - shared write-tool behavior (MCP-bypass principle, silent-ignore rule, envelope, errors)
