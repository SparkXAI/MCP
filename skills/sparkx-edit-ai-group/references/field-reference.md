# Field reference - managed-group create/edit (exact write field names + enums)

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
Rule/RBA. Edit may set a mode field to `0` to switch **RBA -> AI**. It must not set a
mode field to `1` to switch **AI -> RBA**, and it cannot edit RBA conditions/actions.

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

## Edit-only fields - SD `edit_sd_ai_managed_group`

These exist on the SD edit tool (not on create). `*Type` selects a mode that dictates
which value field is valid - see `edit-sd.md` for the full mutual-exclusion table.

| Field | Values |
|---|---|
| `acosType` | `1`=value (-> `acos`), `2`=recommended (-> `acosStatus=1`), `3`=ratio (-> `acosRatio`) |
| `acosStatus` | `1`=apply recommended ACOS |
| `roasType` | `1`=value (-> `roas`), `2`=recommended, `3`=ratio (-> `roasRatio`) |
| `budgetType` | `1`=total (-> `budget`), `2`=daily (-> `budget`), `3`=ratio (-> `budgetRatio`) |
| `campaignNameRecoveryType` | when turning the name label off: `1`=keep current, `2`=restore original |
| `remark` | free text, max 200 chars |
| `ids` | array of aiGroupIds to update, **max 20** |

## SP/SB batch-only fields - inside `request`

When `operation` is non-null, the tool uses `ids` as the target set and ignores
`aiGroupId`. Operation values are **UPPER_SNAKE_CASE** - verified live against the tool
enum (lowercase is rejected with `does not have a value in the enumeration [...]`):

`BUDGET`, `STATUS`, `TARGET_TYPE`, `ACOS`, `ROAS`, `TARGET_HARVEST`, `DYNAMIC_BUDGET`,
`CAMPAIGN_NAME_SIGN`, `AI_PERSONALITY`, `ACTION_SETTINGS`, `BID_OPTIMIZATION`,
`STRUCT_OPTIMIZATION`, `BUDGET_DAYPART`, `BUDGET_DYNAMIC_ACTION`, `BUDGET_REDISTRIBUTE`,
`TARGET_HARVEST_ACTION`, `NEGATIVE_TARGET`, `BRAND_TARGET`, `TARGET_PAUSED_ADD`.

Flat operations read these fields from `batchParams`: `budgetType`, `budget`,
`budgetRatio`, `status`, `optimizeType`, `acosStatus`, `acosType`, `acos`,
`acosRatio`, `roasType`, `roas`, `roasRatio`, `targetHarvestStatus`,
`budgetDynamicStatus`, `numType`, `num`, `campaignNameSign`,
`campaignNameRecoveryType`, and `aiPersonality`. See `batch.md` for the operation to
parameter mapping and mutual-exclusion rules.

## Known value-range caveat

- **`budgetType=3`** (ratio) is a known-risky value - it can hang the downstream
  service. Avoid unless ratio mode is specifically needed; if used, send `budgetRatio`
  and no `budget`.
