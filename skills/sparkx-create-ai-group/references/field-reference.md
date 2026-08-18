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

Each action-space `xxxStatus` is an on/off switch: `0` = off, `1` = on. It does not
identify AI versus Rule/RBA; use the corresponding `aiAutomation` mode field for that.

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

Word-list fields may appear in some schemas, including `brandedStatus`, `brandedList`,
`competitorStatus`, `competitorList`, `negativeTargetBlackListStatus`, and
`targetHarvestBlackListStatus`. They are **currently unsupported**. Do not send any
word-list status, list ID, match-type, or list-type field.

### `aiAutomation` (nested, AI/Rule mode fields) - EXACT names

`aiActionSettings.xxxStatus` controls whether an action space is enabled. When it is
enabled, the corresponding `aiAutomation` field selects the mode: `0` = AI, `1` =
Rule/RBA. Create supports **AI mode only**, so create calls may send `0` but must never
send `1` or attempt to attach an RBA template.

| `aiActionSettings` switch | `aiAutomation` mode field |
|---|---|
| `bidDaypartStatus` | `bidDaypartStatus` |
| `bidPerformanceStatus` | `bidPerformanceRuleStatus` |
| `budgetDaypartActionStatus` | `budgetDaypartRuleStatus` |
| `budgetDynamicActionStatus` | `budgetPerformanceRuleStatus` |
| `negativeTargetActionStatus` | `negativeTargetRuleStatus` |
| `structPauseCampaignStatus` | `pauseCampaignRuleStatus` |
| `bidAdPlaceStatus` | `placementAdjustmentRuleStatus` |
| `targetHarvestActionStatus` | `targetHarvestRuleStatus` |
| `targetPausedAddStatus` | `targetPauseSupplementRuleStatus` |

`budgetRedistributeActionStatus` and `bidAmazonBusinessStatus` have no Rule mode; only
their `aiActionSettings` on/off switch applies.

> Coupling rules (open a switch -> must also send its companion fields) are in
> [`coupling-rules.md`](coupling-rules.md).

## Known value-range caveat

- **Avoid `budgetType=3`** where it appears in edit/action paths - a known P0 that can
  hang the downstream service. Not needed for a basic create.
