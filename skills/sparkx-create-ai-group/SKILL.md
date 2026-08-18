---
name: sparkx-create-ai-group
description: >-
  Create a NEW AI managed group (托管组) for Amazon Sponsored ads and place campaigns
  under it when required. Routes by ad type - SD via create_sd_ai_managed_group (campaigns
  required); SP/SB via
  save_sp_sb_ai_managed_group (create mode). Use when the user wants to create / set up /
  新建 / 建一个 a managed group, or put campaigns under AI management for the first time,
  even without the exact words "managed group". This creates the group itself - it is NOT
  for enabling or editing an existing group (use sparkx-edit-ai-group) or deleting one (use
  sparkx-delete-ai-group).
metadata:
  version: 1.0.0
---

# Create AI Managed Group

Create a new AI managed group. SD creation requires campaigns; SP/SB creation may
create an empty group or include campaigns. This is a
**write, non-idempotent** operation: calling it twice creates two groups. Read
[`references/platform-notes.md`](references/platform-notes.md) once before your
first write - it covers auth, the shared response envelope, the 30s-timeout rule,
and how errors come back (`data.error` + an optional `recoveryHint` that isn't always
populated, so some stay generic).

## The one thing that decides everything: ad type

Creation logic differs by ad type, and they use **different tools**. Your first job
is always to determine the campaign type, then route:

| Ad type | Tool | Read before building the call |
|---|---|---|
| **SD** (Sponsored Display) | `create_sd_ai_managed_group` | [`references/create-sd.md`](references/create-sd.md) |
| **SP / SB** (Sponsored Products / Brands) | `save_sp_sb_ai_managed_group` (leave `aiGroupId` empty -> create mode) | [`references/create-sp-sb.md`](references/create-sp-sb.md) |

How to determine the ad type, in order of preference:
1. The user says it ("SP 托管组", "for my Sponsored Display campaigns").
2. Infer it from the campaigns to be added - look them up and read `campaignType`.
3. If still unknown, **ask the user** - don't guess. A group is created for one ad
   type; picking the wrong tool creates the wrong kind of group.

> SP and SB both go through `save_sp_sb_ai_managed_group`, but a few capabilities
> are SP-only (see `create-sp-sb.md`). SB is not SP - don't copy SP-only fields
> into an SB group.

> **Reads use the full signature.** Every `get_entity_metadata` call requires
> `profileIds`, `entity`, and `userContext` -
> `get_entity_metadata(profileIds=[<id>], entity='campaign', userContext='<why>')`.
> The shorthand `get_entity_metadata(entity='campaign')` fails; always pass all three.

## Workflow (same for every ad type)

1. **Resolve inputs to IDs and codes.**
   - **Campaigns -> internal `campaignId`.** Look them up with
     `get_entity_metadata(profileIds=[...], entity='campaign', userContext='...')`.
     Each row has both `campaignId` (internal auto-increment int) and
     `amazonCampaignId` (Amazon's long string ID). **Always pass `campaignId` into
     `campaignIds`; never `amazonCampaignId`** - the Amazon ID is a ~20-digit value
     that overflows the int32 the tool expects and the call fails. The lookup also
     gives each campaign's `campaignType` (routing) and `aiGroupId` (conflict check).
     - **A `campaignId` from `get_ads_perf` may be the Amazon long ID, not the internal
       one.** Never feed a `campaignId` taken from `get_ads_perf` straight into
       `campaignIds` - first map it through `get_entity_metadata` (match on
       `amazonCampaignId`) to the internal `campaignId`. Only the internal int works.
   - **One profile, one ad type.** A create targets a single `profileId`, and every
     campaign must belong to that profile and share the same `campaignType`. If the
     user mixes profiles or mixes SP + SB (or gives a name that matches campaigns in
     several stores), stop and clarify - don't silently pick a store or mis-route.
   - **Chinese terms -> codes.** Map Chinese goal wording ("推动增长", "活动冲量",
     "保持订单稳定", "激进人格") to the right field + code via
     [`references/enum-i18n.md`](references/enum-i18n.md) before building the call,
     and translate back when you confirm/report.
2. **Pre-flight checks:**
   - **Name uniqueness.** The `aiGroupName` filter is a `like` (substring) match, not
     exact - query `get_entity_metadata(profileIds=[...], entity='aiGroup',
     filters={"aiGroupName": {"like": "%<name>%"}}, userContext='...')`, then compare names
     **exactly** in the returned rows. If an exact match exists, ask for a different
     name rather than letting the create fail.
   - **Name not blank.** Reject a pure-whitespace / trims-to-empty name yourself - the
     UI blocks it, MCP doesn't.
   - **Already-managed campaigns (checkable).** A campaign already in another managed
     group shows a non-empty `aiGroupId` in its metadata - flag those, let the user
     decide.
   - **Budget-management conflict (NOT pre-checkable).** Campaign metadata does not
     expose budget-management membership, so you cannot verify this up front - only
     the backend create can reject it. Don't claim conflicts are fully cleared; say
     the budget-management check happens server-side.
3. **Confirm before creating - show everything that will take effect.** Echo the
   **complete** config, not just the basics: ad type, group name, the campaigns (by
   name), `targetType`/`optimizeType`, target ACOS, budget settings, `aiPersonality`,
   `campaignNameSign`, and **every supported action-space switch you're enabling**
   (bid / budget / target / struct optimization). For anything you're not
   setting, say it will use the platform default - but **don't invent specific default
   values** (the tool schema doesn't define them); only state a concrete default if
   you've read it back or it's documented. Get an explicit go-ahead. This matters most
   when AI will start on (`aiStatus=1`) - those switches immediately affect live
   delivery and spend.
   - **Turn AI on only if the user explicitly asked to start it.** If they said "create
     and start" / "启动", set AI on. If they only asked to set up / create the group or
     place campaigns into it (or didn't say), default AI **off** (`aiStatus` / `status`
     = 0) and state that in your confirmation. Don't start automation on settings the
     user didn't confirm.
4. **Build and call the routed tool** using the exact **write** field names + enum
   values (ad-type reference + `field-reference.md`; write names != read names).
   > **Create is non-idempotent - never blind-retry.** On any failure, timeout, or
   > missing response, the group **may already have been created**. Before retrying,
   > re-query by name (`get_entity_metadata entity='aiGroup'`, exact-match the
   > `smartCreationName`/`aiGroupName`); only create again if it truly does not exist.
   > A blind retry creates a duplicate group.

   **Before sending, self-validate the front-end-only rules MCP bypasses** (see
   platform-notes "MCP bypasses the platform UI's validation"):
   - **Action-space support** - only enable a capability that supports AI for this ad
     type ([`references/action-space-matrix.md`](references/action-space-matrix.md));
     if the user asked for one that isn't supported for their ad type, tell them and
     skip it - don't send a silently-ignored field.
   - **Budget** - sanity-check what you reliably can (positive; JP integer-only;
     multi-campaign minimum scales with campaign count). Exact per-site/account ranges
     are only partly known and the backend may enforce them, so relay backend range
     errors rather than hard-blocking on an incomplete table
     ([`references/budget-limits.md`](references/budget-limits.md)).
   - **aiPersonality** `1`-`5`, and **>=3 when `targetType=3` (volume / 冲量)**.
   - **Coupled fields** - enabling a switch requires its companion fields, and any
     range must have min <= max (see [`references/coupling-rules.md`](references/coupling-rules.md)).
   - **Invalid values are rejected by the backend now - but pre-validate anyway** so the
     user gets a clear message instead of a downstream error (prod-confirmed 2026-08-13):
     `acos` must be > 0 and in range (`0`, negatives, and over-limit are all rejected);
     `aiPersonality` outside `1`-`5` is rejected; `campaignIds` is capped at **1000** per
     group; and a switch sent without its companion field (`acosType` without `acos`,
     `budgetType` without `budget`, dynamic budget without `numType`+`num`) is rejected.
     Catch these up front rather than leaning on the backend error.
   - **Word-list settings are not supported.** Do not send branded, non-branded,
     competitor, harvest-blacklist, or negative-target-blacklist fields, even if the
     routed schema exposes them. Tell the user to configure word lists in the platform.
5. **Verify it landed - group AND campaigns.** Don't trust the envelope alone:
   - Re-read the group (`entity='aiGroup'`) -> confirm it exists with the intended
     top-level settings.
   - Re-read `entity='campaign'` and check each intended campaign's `aiGroupId` now
     points at the new group - the group can exist with fewer campaigns than you sent.
   - **SP/SB: a `success` response can still leave an EMPTY group.** `save_sp_sb_ai_managed_group`
     may create the group without immediately binding `campaignIds`. So after create,
     count how many of your campaigns actually have `aiGroupId` = the new group. **If the
     bound count is 0 (or short), re-attach via edit mode**: call `save_sp_sb_ai_managed_group`
     again with the returned `aiGroupId` (> 0) and the full `campaignIds`, then re-verify.
   - **Reading back binding: query per campaign, not one big `in` list.** In some
     environments an `campaignId: {"in":[...]}` filter throws a backend type error.
     If a batched filter errors, fall back to per-id `campaignId = <id>` reads (or another
     confirmed-usable field). This applies to both the pre-create mapping and this check.
   - Nested `aiActionSettings`/`aiAutomation` values may not be independently
     readable; if you can't read one back, say "created, but this setting couldn't be
     independently confirmed" rather than implying it's verified.
   - If operation-log read access is available, query `get_operation_log` and confirm
     the create is recorded against an identifiable token user. The server supplies
     `changedBy`; never send or fabricate it. If logs cannot be read, state that audit
     verification was not performed.

The current write schemas do not expose managed-group scheduling fields. Do not invent
`scheduleType` or schedule-date parameters; scheduling must be done in the platform.

The current tools also cannot create a managed group from a platform template. Build
the supported configuration explicitly from the user's requirements, or tell the user
to use the platform when a template-based setup is required.

## AI on vs off at creation

`aiStatus` (SP/SB) / `status` (SD) decides whether AI starts optimizing immediately.
Turn it on **only when the user explicitly asked to create-and-start**. If they just
want the group set up (or didn't say), default to AI **off** and say so in your
confirmation - turning AI on means it starts adjusting bids/budgets and spending against
the target right away.

> Note: an "off" group reads back as `aiStatus=2` ("AI Turned Off"), not `0`. Don't
> treat a non-zero `aiStatus` on read as "it's on" - `1` = running, `2` = off.

## Enum discipline

`optimizeType` / `targetType` / `status` / `aiStatus` / `targetHarvestStatus` /
`budgetType` / `aiPersonality` and the `aiActionSettings` switches are closed enums.
Use only the values listed in
[`references/field-reference.md`](references/field-reference.md) - do not invent or
infer values. Passing an unlisted value makes the call fail, and the error comes back
generic (no field-level hint), so validate against the dictionary **before** sending
rather than relying on the error to tell you what was wrong. For mapping the user's
Chinese wording to these codes (and back), use
[`references/enum-i18n.md`](references/enum-i18n.md).

## Response & errors

Success looks like `{ "isError": false, "data": { "status": "success", "result": {
"aiGroupId": <new id> } } }` - capture `aiGroupId` for the verify step.

Errors come back as `{ "isError": true, "data": { "error": "...", "recoveryHint":
"..." } }`. Relay `recoveryHint` when present, but it isn't always populated - some
come back generic - so still map the common ones yourself:

| Symptom | Likely cause | What to do |
|---|---|---|
| duplicate-name / "Duplicate group name" | name already used in this profile | ask for a different name |
| "Campaign conflict" | a campaign is already in another AI group / budget mgmt | surface which campaign; drop it or free it first |
| generic `business_error` | often an out-of-range/invalid field value | re-check every value against `field-reference.md` |
| timeout (~30s, no response) | the create may have already applied downstream | **verify with a read before retrying** - a blind retry can create a duplicate group |

## Reference files
- [`references/create-sd.md`](references/create-sd.md) - SD create (`create_sd_ai_managed_group`) fields + example
- [`references/create-sp-sb.md`](references/create-sp-sb.md) - SP/SB create (`save_sp_sb_ai_managed_group`): core fields, SP-vs-SB differences, action-space **coupling rules** + example
- [`references/field-reference.md`](references/field-reference.md) - exact **write** field names + enum values (write names != read names)
- [`references/action-space-matrix.md`](references/action-space-matrix.md) - which action-space capabilities are supported (AI / Rule / none) per SP / SB / SD
- [`references/budget-limits.md`](references/budget-limits.md) - site/account-type budget ranges (front-end rules MCP bypasses)
- [`references/enum-i18n.md`](references/enum-i18n.md) - 中文 <-> English <-> code mapping (parse Chinese requests -> codes; render codes -> Chinese)
- [`references/platform-notes.md`](references/platform-notes.md) - shared write-tool behavior (auth, response envelope, timeout, error shape)
