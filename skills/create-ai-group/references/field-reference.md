# Field reference - managed-group create (exact write field names + enums)

All coded fields are **closed enums**; an unlisted value fails the call with a generic
error and no field-level hint, so validate here before sending. Field **names** also
matter: the tools set `additionalProperties: false`, so a wrong or extra field name is
rejected outright.

> **Read names != write names.** The names `get_entity_metadata` *returns* are not
> always the names the write tools *accept*. E.g. the read side shows
> `budgetDaypartStatus`, but SP/SB writes require **`budgetDaypartActionStatus`**.
> The "Action" suffix is present on some action-space switches and absent on others,
> and SD-create fields differ from SP/SB fields. **When building a write call, use the
> exact names in this file (which mirror the tool inputSchemas), never the read-side
> field names.**

## Shared enums

| Field | Values |
|---|---|
| `aiPersonality` | `1`=非常保守, `2`=保守, `3`=平衡, `4`=激进, `5`=非常激进. **>=3 required when `targetType=3` (volume/冲量)** - front-end rule, not backend-enforced |
| `campaignNameSign` | `0`=off, `1`=on |
| `numType` / `budgetNumType` / `bidRangeType` | `1`=percentage, `2`=fixed value |
| `*MatchType` (branded/competitor/harvest/negative) | `1`=exact(等于/精确), `2`=phrase(包含/词组) |
| `*ListType` (harvest/negative) | `1`=include/whitelist, `2`=exclude/blacklist |

## SD - `create_sd_ai_managed_group` fields

Flat fields (no nested objects):

| Field | Values / type |
|---|---|
| `status` | **create: `0`=off, `1`=on only** (`2`=cancelled is a lifecycle state, **not** a valid create input) |
| `optimizeType` | `1`=drive growth, `2`=maintain stability |
| `acos` | number, ACOS on the x100 scale (see create-sd.md) |
| `budget` / `budgetChange` | number / boolean (budget applies only when `budgetChange=true`) |
| `budgetDynamicStatus` | `0`/`1` |
| `numType` / `num` | value type / value for dynamic budget |
| `targetHarvestStatus` | `0`=off, `1`=on, `2`=on with exact negation in source ad group |
| `budgetRedistributeStatus` | `0`/`1` |
| `campaignNameSign` | `0`/`1` |
| `aiPersonality` | `1`-`5` |

## SP/SB - `save_sp_sb_ai_managed_group` `request` fields

Top-level: `aiStatus` (`0`=off, `1`=on - create input is `0`/`1`; a group reads back
as `2`=paused once off), `targetType` (`1`=drive growth, `2`=maintain stability,
`3`=volume, `4`=legacy growth), `campaignType` (`sponsoredProducts`/`sponsoredBrands`),
`acos`, `campaignIds`, `campaignNameSign`, `aiPersonality`, `preAddCampaignNums`.

### `aiActionSettings` (nested) - EXACT names

Bid: `bidDaypartStatus`, `bidPerformanceStatus`, `bidPerformanceStrictAcosStatus`,
`bidAmazonBusinessStatus`, `bidAdPlaceStatus`, `bidAdPlaceRangeStatus`,
`bidRangeStatus`, `bidRangeType`, `bidRange` (`[min,max]`, null=no limit),
`btbRangeStatus`, `btbMin`, `btbMax`, `tosMin`/`tosMax`, `pdpMin`/`pdpMax`,
`rosMin`/`rosMax`.

Struct (**SP only**): `structPauseProductStatus`, `structPauseCampaignStatus`.

Budget: **`budgetDaypartActionStatus`**, **`budgetDynamicActionStatus`**,
**`budgetRedistributeActionStatus`**, `budgetNum`, `budgetNumType`.

Target: `targetHarvestActionStatus`, `targetHarvestBlackListStatus`,
`targetHarvestBlackList` (IDs), `targetHarvestListType`, `targetHarvestMatchType`,
`negativeTargetActionStatus`, `negativeTargetBlackListStatus`,
`negativeTargetBlackList` (IDs), `negativeTargetListType`, `negativeTargetMatchType`,
`targetPausedAddStatus` (`0`=off, `1`=on, `2`=on with supplement).

Brand: `brandedStatus`, `brandedMatchType`, `brandedList` (IDs); `competitorStatus`,
`competitorMatchType`, `competitorList` (IDs).

`brandedStatus` / `competitorStatus` (+ their lists) are **not** version-gated - the
backend doesn't filter by group version (v1/v2 is UI-only); prod-confirmed (2026-08-13)
they're accepted and take effect on v1, so send them normally (list IDs user-supplied).
But the **blacklist word-lists** `negativeTargetBlackListStatus` /
`targetHarvestBlackListStatus` are **currently unsupported for every ad type** (2026-08-14
spec: should be rejected) - **do not send them**; the backend may not block it yet but it
has no effect.

### `aiAutomation` (nested, mode switches) - EXACT names

When an action space is on (`aiActionSettings.xxxStatus = 1`), the corresponding
`aiAutomation` field controls the mode: `0` = AI mode (AI auto-decision),
`1` = Rule mode (condition/action template governs). Field names vary — not all
follow the `xxxRuleStatus` pattern.

**Action-space switch -> aiAutomation mode field mapping:**

| `aiActionSettings` switch | `aiAutomation` mode field | rule# |
|---|---|---|
| `bidDaypartStatus` | `bidDaypartStatus` | 2 |
| `bidPerformanceStatus` | `bidPerformanceRuleStatus` | 181 |
| `budgetDaypartActionStatus` | `budgetDaypartRuleStatus` | 13 |
| `budgetDynamicActionStatus` | `budgetPerformanceRuleStatus` | 17 |
| `negativeTargetActionStatus` | `negativeTargetRuleStatus` | 5 |
| `structPauseCampaignStatus` | `pauseCampaignRuleStatus` | 20 |
| `bidAdPlaceStatus` | `placementAdjustmentRuleStatus` | 19 |
| `targetHarvestActionStatus` | `targetHarvestRuleStatus` | 4 |
| `targetPausedAddStatus` | `targetPauseSupplementRuleStatus` | 182 |

`budgetDaypartExcuteDays` = comma-separated days (`1`-`6`=Mon-Sat, `0`=Sun;
default `"1,2,3,4,5,6,0"`). At create time, defaults are AI mode (`0`) unless
the user specifies otherwise.

`noRule` capabilities (`budgetRedistributeActionStatus`, `bidAmazonBusinessStatus`)
have **no** `aiAutomation` mode field — only on/off via `aiActionSettings`.

> Coupling rules (open a switch -> must also send its companion fields) are in
> [`coupling-rules.md`](coupling-rules.md).

## Known value-range caveat

- **Avoid `budgetType=3`** where it appears in edit/action paths - a known P0 that can
  hang the downstream service. Not needed for a basic create.
