# Field Reference — `get_entity_metadata`

## `entity` Enum Values (and backing provider)

| entity | Backing provider | Notes |
|---|---|---|
| `profile` | AdsListMetadataProvider | Store/profile list. **Can be queried standalone** — it is a first-class entity here, not a required-join-only dimension |
| `campaign` | AdsListMetadataProvider | Campaign list |
| `adGroup` | AdsListMetadataProvider | Ad group list |
| `target` | AdsListMetadataProvider | Targeting list |
| `productAd` | AdsListMetadataProvider | Product ad list |
| `portfolio` | AdsListMetadataProvider | Portfolio list |
| `placement` | AdsListMetadataProvider | Placement list |
| `aiGroup` | AiGroupMetadataProvider | AI managed group list. Response is a **projection of the currently effective config** — see SKILL.md |
| `aiGroup_schedule` | AiGroupScheduleMetadataProvider | Schedules of **one** managed group — **requires exactly one `profileId` and `filters.aiGroupId` only**, ignores pagination/sorting |
| `asin` | AsinMetadataProvider | ASIN product info (child ASIN + parent ASIN + product line, nested) |
| `automationRule` | AutomationRuleMetadataProvider | Enabled rule-type codes/names for given campaign(s) — **requires `amazonCampaignId` in filters; does not return template configuration** |

## Fields by Entity

### profile

**Return fields**: `profileId`, `profileName`, `countryCode` (enum: `US`/`CA`/`MX`/`UK`/`DE`/`FR`/`IT`/`ES`/`JP`/`NL`/`AU`/`SG`/`BR`/`SE`/`AE`/`PL`/`IN`/`TR`/`SA`/`BE`/`EG`/`ZA`), `profileDailyBudgetCap`, `profileUseBudgetCap`

**Filterable**: `profileName` (`like` only) — e.g. `{"profileName": {"like": "%star%"}}`

### campaign

**Every campaign return field is filterable** - per the sparkxads API, for the campaign
entity the filterable set = **all return fields, no exceptions** (so `aiGroupId`,
`portfolioId`, `campaignStartDate`/`campaignEndDate`, and the `campaignAi*Date` fields can
all be used in `filters`, not just the core fields below). The two tables are split for
readability only.

**Core fields:**

| Field | Type | Enum |
|---|---|---|
| campaignId | int | SparkX AI internal auto-increment campaign ID; use this value for managed-group write-tool `campaignIds` |
| amazonCampaignId | string | Amazon campaign ID; use this value to link performance/log data, not as a managed-group write ID |
| campaignName | string | — |
| campaignType | string | `sponsoredProducts` / `sponsoredBrands` / `sponsoredDisplay` |
| campaignState | string | `enabled` / `paused` / `archived` |
| biddingStrategy | string | `legacyForSales` / `autoForSales` / `manual` / `ruleBased` |
| targetingType | string | `auto` / `manual` |
| costType | string | `cpc` / `vcpm` |
| isAiCreate | int | `1` / `0` |
| dailyBudget | number | — |
| currentBudget | number | — |

**Campaign ID contract:** `campaignId` and `amazonCampaignId` are different identifiers.
Managed-group create/edit tools accept the internal integer `campaignId`. When starting
from an Amazon campaign ID returned by `get_ads_perf` or `get_operation_log`, filter this
entity by `amazonCampaignId`, then use the matched row's internal `campaignId`. Never
coerce a long Amazon ID into the write field or infer one identifier from the other.

**⚠️ "How much budget is left today?" cannot be reliably answered from `dailyBudget`/`currentBudget` alone.** These are configuration values, not a live spend ledger, and can change intraday. Combining them with `get_ads_perf`'s today's-`Spend` doesn't produce a reliable real-time remaining-budget figure either, since that metric is subject to the T+2 processing delay (see PLATFORM_NOTES.md). Tell the user this can't be computed reliably rather than presenting a subtraction as if it were precise.

**More campaign fields (also filterable):**

| Field | Type | Enum |
|---|---|---|
| campaignStartDate | string (Ymd) | — |
| campaignEndDate | string (Ymd) | — |
| portfolioId | string | — |
| aiGroupId | string | — |
| campaignAiFirstOnDate | string | — |
| campaignAiLastOnDate | string | — |
| campaignAiLastOffDate | string | — |

**⚠️ `campaignStartDate`/`campaignEndDate` use `YYYYMMDD` (Ymd) format — e.g. `"20260101"`, not `"2026-01-01"`.** This is different from the `dateStart`/`dateEnd` request parameters on the other two tools, which use `YYYY-MM-DD`. This tool has no `dateStart`/`dateEnd` params of its own — these are just two ordinary campaign config fields that happen to hold dates, in Ymd format.

**Filter examples** (each is a standalone example of a `filters` value — not one combined call):
```json
{"campaignState": {"in": ["enabled", "paused"]}}
```
```json
{"campaignType": {"in": ["sponsoredProducts"]}}
```
```json
{"dailyBudget": {">=": 10, "<=": 100}}
```
```json
{"campaignName": {"like": "%brand%"}}
```
```json
{"amazonCampaignId": {"in": ["298539385213868"]}}
```
```json
{"campaignStartDate": {">=": "20260101", "<=": "20260131"}}
```
```json
{"biddingStrategy": "autoForSales"}
```

### adGroup

| Field | Type | Enum |
|---|---|---|
| adGroupId | string | — |
| campaignId | string | — |
| adGroupName | string | — |
| adGroupState | string | `enabled` / `paused` / `archived` |
| defaultBid | number | — |
| sdBidOptimization | string | `clicks` / `conversions` / `reach` (SD only) |

### target

| Field | Type | Enum |
|---|---|---|
| targetId | string | — |
| adGroupId | string | — |
| campaignId | string | — |
| targetText | string | — |
| targetMatchType | string | Keyword: `exact`/`phrase`/`broad`. Product: `asinSameAs`/`asinExpandedFrom`/`asinCategorySameAs`. Auto: `queryHighRelMatches`/`queryBroadRelMatches`/`asinSubstituteRelated`/`asinAccessoryRelated`/`similarProduct` |
| targetState | string | `enabled` / `paused` / `archived` |
| **targetBid** | number | Current bid — use this for "bid > $X" filters |

**⚠️ No documented way to query the current list of negative keywords/ASINs/brands.** None of `targetMatchType`'s enum values above represent a negative-targeting variant, on this tool or on `get_ads_perf`'s equivalent field. `get_operation_log` can tell you *when* a negative target was added/removed (via its `targetTypes` filter values `negativeKeyword`/`negativeAsin`/`negativeBrand`), but that's change history, not a queryable current snapshot. If a customer asks "show me my negative keywords," this is a genuine tool capability gap, not something to work around with a clever filter — say so, don't guess at an undocumented field or match-type value.

### productAd

| Field | Type | Enum |
|---|---|---|
| amazonAdId | string | — |
| adGroupId | string | — |
| campaignId | string | — |
| asin | string | — |
| sku | string | — |
| productAdState | string | `enabled` / `paused` / `archived` |

### portfolio

| Field | Type | Enum |
|---|---|---|
| portfolioId | string | — |
| portfolioName | string | — |
| portfolioState | string | `enabled` / `paused` / `archived` |
| portfolioServingStatus | string | `IN_BUDGET` / `OUT_OF_BUDGET` / `PORTFOLIO_ENDED` |
| portfolioStartDate | string | — |
| portfolioEndDate | string | — |
| **portfolioBudget** | number | Budget amount — use this for "which portfolios have a budget set" (filter `{"portfolioBudget": {">": 0}}`) |
| portfolioBudgetType | string | `dateRange` / `monthlyRecurring` |

### placement

One row per campaign × placement combination.

| Field | Type | Enum |
|---|---|---|
| campaignId | string | — |
| placement | string | `topOfSearch` / `productPage` / `restOfSearch` |
| multiplier | number | Bid adjustment % |

### aiGroup (AI Managed Group)

Sourced from a separate backend (`td-api getSaAiGroupList`), aggregated from the SA perspective — richer field set than the ads DB.

**Return fields** (partial — full list is long):
`aiGroupId`, `aiGroupName`, `aiStatus` (`0`=AI never turned on, `1`=AI currently running, `2`=AI turned off), `campaignType`, `targetType` (`1`=Drive Growth/`2`=Maintain Stable Orders/`3`=Event Boost), `targetAcos`, `aiPersonality` (1-5), `aiPersonalityUpdatedAt`, `profileId`, `profileName`, `countryCode`, `numCampaign`, `numProduct`, `campaignNameSign`, `createTime`, `createBy`, `createUid`, `hasEditAuth`, `isAutoPacing`, `statusOnDate`, `lastStatusOnDate`, `lastStatusOffDate`, `lastOnDays`, `lastOnDaysBegin`, `lastOnDaysEnd`, `totalBudget`, `totalDailyBudget`, `sbStyleNum`, `aiActionSettings`, `aiAutomation`

- **`sbStyleNum`** (int/null): the **count** of SB ad styles in use — e.g. "3" means 3 different SB ad style/formats are running. This can answer "how many SB ad styles is this group using," but **not** "which specific style(s)" (product collection / store spotlight / video, etc) — that breakdown is not exposed by this field and is not otherwise documented in the platform spec. Don't infer or invent a style name from the count.
- **`totalBudget` / `totalDailyBudget`** (number): the **sum of the group's enabled
  campaigns' daily budgets** (i.e. 托管组总预算 / group total budget). These are **read-only
  rollups** here. On the write side, editing the group total **proportionally rescales
  every enabled campaign's daily budget** to the new total (see the create/edit skills) —
  don't treat them as independent editable fields.
- **`aiActionSettings`** (object): action-space switches and their parameters. A relevant
  `xxxStatus` value of `0` means off and `1` means on.
- **`aiAutomation`** (object): keyed by rule type (`2`, `4`, `5`, `13`, `17`, `19`, `20`,
  `181`, `182`). Each entry's `status` is the mode: `0` = AI, `1` = Rule/RBA. Pair an
  action-space switch with its mode entry rather than deciding the mode from
  `aiActionSettings` alone. Conversely, do not decide from `aiAutomation` alone either:
  an empty object can mean the paired action spaces are off, that enabled action spaces are
  AI-driven, or both. Read the paired switch first.
- **⚠️ Both objects are returned as a projection of what's *currently effective*, not the
  raw stored record.** Fields belonging to a switch that is off are omitted; rules running
  in AI mode are omitted from `aiAutomation` entirely; rules in Rule mode drop their
  AI-only parameters; and the whole set is first trimmed to what the `campaignType`
  supports. **An absent field means "not in effect", not "unset" and not "write failed".**
  Full rules and the per-ad-type trim table are in SKILL.md — read them before reporting a
  config or verifying a write.
- **RBA rule configuration is readable.** For a rule in Rule mode, `aiAutomation.{ruleType}`
  carries its real conditions, actions, condition items, time periods, and (for dayparting)
  the hour matrix. Confirmed leaves carry a `...Text` companion; unconfirmed leaves are
  passed through raw with no Text — relay those verbatim rather than labelling them. The
  same raw value can mean different things under different rule types, so never carry a
  label across rules. Reading is supported; **writing RBA config is not available through
  any MCP tool**.

For the paired-switch table, standalone-vs-managed capability boundary, condition priority,
and rule-specific business semantics, read [`automation-rule-reading.md`](automation-rule-reading.md)
before explaining the configuration to a user.

**Automation rule <-> action-space mapping** (when a managed-group action space is in
Rule/RBA mode, its readable config appears under `aiAutomation.{ruleType}`):

| action space | ruleType | automation rule template | Notes |
|---|---:|---|---|
| `bidDaypart` / 分时调价 | `2` | Bid dayparting / 分时调价 | Standalone automation supports SP/SB/SD; managed-group action-space support follows the action-space matrix. |
| `targetHarvest` / 定向收割 | `4` | Harvest Keywords / 添加搜索词 | `targetHarvestStatus` special source-exact-negative modes are represented on the action-space switch, not by a different ruleType. |
| `negativeTarget` / 添加否定定向 | `5` | Add Negative Keywords / 添加否定词 | SP/SB standalone automation; SP/SB managed-group word-list settings are read-only through MCP. |
| `budgetDaypart` / 分时预算 | `13` | Budget dayparting / 分时预算 | SP/SB standalone automation. |
| `budgetPerformance` / 按表现调预算 | `17` | Budget rules / 预算规则 | Help Center calls this Budget Rules; action-space docs also call it budget by performance / boost budget. |
| `bidAdPlace` / 广告位调价 | `19` | Placement Rules / 广告位规则 | Help Center says standalone Placement Rules support SP/SB; managed-group action-space use may still be SP-only per the action-space matrix and PRD template filtering. |
| `structPauseCampaign` / 暂停广告活动、广告组 | `20` | Campaign activation/pausing (New) / 开启/暂停广告活动（新版） | Replaces the old pause/enable campaign rules for new setup. |
| `bidPerformance` / 按表现调价 | `181` | Bid by performance / 按表现调价 | Managed-group action-space RBA rule type. |
| `targetPausedAdd` / 定向暂停/补充 | `182` | Target pause/supplement / 定向暂停/补充 | Managed-group action-space RBA rule type. |

`budgetRedistribute` (预算重新分配), `bidAmazonBusiness` (B2B 调价), and
`structPauseProduct` (暂停商品) are action spaces without an `aiAutomation` ruleType in
the current MCP projection.

**Filterable fields — this is a closed whitelist.** Unlike `campaign` (where every returned
field is filterable), `aiGroup` accepts only the fields below. Anything else fails with
`Invalid aiGroup filter 'x': is not supported for aiGroup`, and an out-of-domain enum value
fails too (e.g. `campaignType` must be one of `sponsoredProducts` / `sponsoredBrands` /
`sponsoredDisplay`; `targetType` must be `1`/`2`/`3`). Don't assume a field is filterable
just because it appears in the response.

| Field | Type | Format | Example |
|---|---|---|---|
| aiStatus | int/array | value or `in` | `{"aiStatus": 1}` or `{"aiStatus": {"in": [0, 1, 2]}}` |
| campaignType | array | `in` | `{"campaignType": {"in": ["sponsoredProducts"]}}` |
| aiGroupId | array | `in` | `{"aiGroupId": {"in": [123, 456]}}` |
| aiGroupName | string | `like` | `{"aiGroupName": {"like": "%keyword%"}}` |
| targetType | string/array | `in` or comma | `{"targetType": {"in": ["1", "2"]}}` |
| targetAcos | number | range | `{"targetAcos": {">=": 10, "<=": 50}}` |
| portfolioId | array | `in` | `{"portfolioId": {"in": [123, 456]}}` |

**`targetAcos` uses the same confirmed ×100/percentage scale as performance `ACOS` in `get_ads_perf`** (documented as "Target ACOS (percentage)") — a value of `35` means a 35% target. Don't re-scale the number, but **append `%`** when presenting it: "target ACOS is 35%".

**orderBy supported fields**: `aiGroupName`, `createTime` (default), `createBy`

### aiGroup_schedule (managed-group schedules)

Schedules ("flights") of **one** managed group. Special calling contract — see SKILL.md and [`example-ai-group-schedule.md`](example-ai-group-schedule.md).

**Request contract** (all enforced, all fail-closed):

| Requirement | Error when violated |
|---|---|
| Exactly one authorized `profileId` | `aiGroup_schedule query requires exactly one authorized profileId` |
| `filters` contains `aiGroupId` | `aiGroup_schedule query requires filters.aiGroupId` |
| `filters` contains **nothing else** | `aiGroup_schedule query only supports filters.aiGroupId` |
| `aiGroupId` is a single positive integer (`29123`, `[29123]`, or `{"in": [29123]}`) | invalid-filter error |

Pagination and `orderBy` are ignored; all schedules for the group come back in one response, and the top-level `page`/`pageSize`/`hasNextPage` fields are omitted.

**Return fields** (per schedule row):

| Field | Type | Notes |
|---|---|---|
| `id` | long | Schedule ID (used as the update/delete key by the write tool) |
| `isActive` | boolean | Whether the schedule is active |
| `timeType` | int | `1` = fixed date window, `2` = weekly repeat |
| `startDate` / `endDate` | string | `timeType=1` only |
| `weekDays` | array[int] | `timeType=2` only. `1`=Monday … `7`=Sunday |
| `optimizeType` | int | `1`=Drive Growth, `2`=Maintain Stable Orders, `3`=Event Boost, `4`=Drive Growth · Budget-utilization priority. **`timeType=2`: inherited from the parent group, read it from the group row instead** |
| `acos` | number | Target ACOS. Same inheritance note as `optimizeType` |
| `aiPersonality` | int | 1–5. Same inheritance note as `optimizeType` |
| `aiActionSettings` | object | Per-schedule action-space settings, same shape and same effective-config projection as on the group |
| `aiAutomation` | object | Per-schedule automation rules, keyed by rule type, same projection rules |

**Reporting rule**: for a `timeType=2` (weekly) schedule, do **not** report `optimizeType`/`acos`/`aiPersonality` from the schedule row — they're inherited. Read them from the parent group (`entity: aiGroup`) and say so.

**⚠️ `weekDays` read vs write encodings differ.** The values above (`1`=Monday … `7`=Sunday) are what the **read** side (`get_entity_metadata`) returns. The **write** tool `save_sp_sb_ai_group_schedule` expects a **different** encoding — `0`=Sunday … `6`=Saturday. Do **not** echo a `weekDays` value read here straight into a save-schedule call; translate between the two encodings first.

### asin (ASIN product info)

Single query returns **child ASIN + parent ASIN + product line info nested together**. By default only returns non-deleted ASINs (`asinIsDelete=0`).

**Child ASIN fields**: `profileId`, `asin`, `sku`, `parentAsin` (aka virtual_parent_asin), `asinTitle`, `asinBrand`, `asinOpenDate` (SP go-live date), `asinCategoryInfo` (JSON), `asinBsr`, `asinPrice`, `asinFbaQuantity`, `asinInventoryStatus`, `asinSpEligibilityStatus`, `asinSbEligibilityStatus`, `asinSdEligibilityStatus`

**Parent ASIN fields**: `parentAsinTitle`, `parentAsinBrand`, `parentAsinPrice`, `parentAsinBsr`, `parentAsinInventoryStatus`, `parentAsinOpenDate`, `parentAsinCategoryInfo`, `parentAsinSpEligibilityStatus`, `parentAsinSbEligibilityStatus`, `parentAsinSdEligibilityStatus`, `parentAsinFbaQuantity`

**Product line fields** (nested array `productLines`, one ASIN may belong to multiple product lines): `productLineParentId`, `productLineParentName`, `productLineChildId` (null = directly attached to parent line, no sub-tag), `productLineChildName`, `productLineCreator`

**Currency**: `asinPrice`/`parentAsinPrice` behave differently for single vs multi-profile queries:

| Scenario | Outer `currency` | Per-row `currency` field | Notes |
|---|---|---|---|
| Single profile | Local currency code | none | `asinPrice`/`parentAsinPrice` in local currency |
| Multi profile | **not present** | **each row carries a `currency` field** | Product pricing is not FX-converted; each row's currency is identified individually |

Multi-profile example:
```json
{
  "isError": false,
  "toolName": "get_entity_metadata",
  "rows": [
    {"asin": "B0XX", "asinPrice": 29.99, "currency": "USD", "profileId": 111},
    {"asin": "B0YY", "asinPrice": 2980, "currency": "JPY", "profileId": 222}
  ]
}
```
This is the one entity where multi-profile does NOT mean USD — check each row's `currency` field individually.

**Filterable fields**:

| Field | Type | Op | Example |
|---|---|---|---|
| asin | string/array | `in` | `{"asin": {"in": ["B0XX", "B0YY"]}}` |
| parentAsin | string/array | `in` | `{"parentAsin": {"in": ["B0PARENT1"]}}` |
| sku | string/array | `in` | `{"sku": {"in": ["SKU001", "SKU002"]}}` |
| asinBrand | string | `like` / `=` | `{"asinBrand": {"like": "%Samsung%"}}` |
| asinTitle | string | `like` | `{"asinTitle": {"like": "%wireless%"}}` |
| asinBsr | number | `>=`, `<=` | `{"asinBsr": {">=": 1, "<=": 1000}}` |
| asinPrice | number | `>=`, `<=` | `{"asinPrice": {">=": 10.0, "<=": 50.0}}` |
| asinInventoryStatus | string/array | `in` | `{"asinInventoryStatus": {"in": ["IN_STOCK"]}}` |
| asinSpEligibilityStatus / asinSbEligibilityStatus / asinSdEligibilityStatus | string | `=` | `{"asinSpEligibilityStatus": "ELIGIBLE"}` |
| asinIsDelete | number | `=` | `{"asinIsDelete": 1}` (to see deleted ones) |
| productLineParentId / productLineChildId | number/array | `in` | `{"productLineParentId": {"in": [101, 102]}}` |
| productLineParentName | string | `like` / `=` | — |

**orderBy supported fields**: `asin`, `asinTitle`, `asinBrand`, `asinBsr`, `asinPrice`, `asinFbaQuantity`, `parentAsin`

### automationRule

Queries which automation rule types are enabled for given campaign(s). **Must pass `amazonCampaignId` in filters — this is not optional for this entity.**

This entity is deliberately narrow: it returns an enabled-type summary, not the account's
automation-template library. It does not expose template ID/name, applicable objects,
conditions, actions, frequency, date windows, or campaign-template associations. To read a
managed group's embedded Rule-mode configuration, query `entity: aiGroup` and inspect
`aiAutomation`; that still does not expose standalone template details.

Product inventory rules are SKU/product-scoped and have no rule type in this campaign lookup.
Their absence from `enabledRuleTypes` does not prove that no product inventory rule exists.

**Return fields**: `amazonCampaignId` (long), `enabledRuleTypes` (array[int] — enabled rule type codes), `enabledRuleNames` (array[string] — human-readable rule names, already localized; there is no separate `Text` companion field for this one since the values are already human-readable)

**ruleType enum** (meaning of the ints in `enabledRuleTypes`):

| ruleType | Name | Description |
|---|---|---|
| 2 | Dayparting | Bid dayparting |
| 3 | Budget rules (legacy) | Deprecated legacy budget rule; may still appear on historical campaigns |
| 4 | Harvest Keywords | Auto-harvest search terms into keywords |
| 5 | Harvest Negative Targeting | Auto-add negative targets |
| 6 | Pause Campaign | Auto-pause campaign |
| 7 | Enable Campaign | Auto-enable campaign |
| 8 | Campaign (pause/enable) | Combined pause/enable rule |
| 13 | Budget Day Parting | Budget dayparting |
| 15 | Target | Targeting rule |
| 17 | Budget Performance | Performance-based budget rule |
| 18 | Target (new) | Targeting rule v1 |
| 19 | Placement Rule | Placement adjustment rule |
| 20 | Campaign V2 (pause/enable) | Combined pause/enable rule V2 |
| 181 | Bid Performance | Managed-group bid-by-performance action-space rule |
| 182 | Target Pause/Supplement | Managed-group target pause/supplement action-space rule |

**Filter**: only `amazonCampaignId` (required)
```json
{"amazonCampaignId": {"in": [123456789, 987654321]}}
```
Shorthand also accepted:
```json
{"amazonCampaignId": [123456789, 987654321]}
```

**Note**: `automationRule` does **not** support sort or pagination — results are returned in the **same order as the `amazonCampaignId` list you passed in `filters`** (not sorted by ID value, and unrelated to `orderBy`/`page`, which have no effect here). It also has no outer `currency` field (rule configuration is not a monetary value).

**Full example**:
```json
{
  "profileIds": [4404871489220462],
  "entity": "automationRule",
  "filters": {"amazonCampaignId": {"in": [123456789, 987654321, 555555555]}},
  "userContext": "Which automation rules are enabled on these campaigns"
}
```
Response:
```json
{
  "isError": false,
  "toolName": "get_entity_metadata",
  "rows": [
    {"amazonCampaignId": 123456789, "enabledRuleTypes": [2, 4], "enabledRuleNames": ["Dayparting", "Harvest Keywords"]},
    {"amazonCampaignId": 987654321, "enabledRuleTypes": [], "enabledRuleNames": []},
    {"amazonCampaignId": 555555555, "enabledRuleTypes": [17, 19], "enabledRuleNames": ["Budget Performance", "Placement Rule"]}
  ],
  "rowCount": 3,
  "page": 1,
  "pageSize": 3,
  "hasNextPage": false,
  "effectiveProfileIds": [4404871489220462]
}
```

---

**Enum i18n**: When presenting enum values to the user or translating between API values and display labels, consult [`enum-i18n.md`](enum-i18n.md) for the complete ZH/EN/JA mapping of all enum fields documented above.
