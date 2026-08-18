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
| `aiGroup` | AiGroupMetadataProvider | AI managed group list |
| `asin` | AsinMetadataProvider | ASIN product info (child ASIN + parent ASIN + product line, nested) |
| `automationRule` | AutomationRuleMetadataProvider | Which automation rules are enabled on given campaign(s) — **requires `amazonCampaignId` in filters** |

## Fields by Entity

### profile

**Return fields**: `profileId`, `profileName`, `countryCode` (enum: `US`/`CA`/`MX`/`UK`/`DE`/`FR`/`IT`/`ES`/`JP`/`NL`/`AU`/`SG`/`BR`/`SE`/`AE`/`PL`/`IN`/`TR`/`SA`/`BE`/`EG`/`ZA`), `profileDailyBudgetCap`, `profileUseBudgetCap`

**Filterable**: `profileName` (`like` only) — e.g. `{"profileName": {"like": "%star%"}}`

### campaign

**Return = filterable fields**:

| Field | Type | Enum |
|---|---|---|
| campaignId | string | — |
| campaignName | string | — |
| campaignType | string | `sponsoredProducts` / `sponsoredBrands` / `sponsoredDisplay` |
| campaignState | string | `enabled` / `paused` / `archived` |
| biddingStrategy | string | `legacyForSales` / `autoForSales` / `manual` / `ruleBased` |
| targetingType | string | `auto` / `manual` |
| costType | string | `cpc` / `vcpm` |
| isAiCreate | int | `1` / `0` |
| dailyBudget | number | — |
| currentBudget | number | — |

**⚠️ "How much budget is left today?" cannot be reliably answered from `dailyBudget`/`currentBudget` alone.** These are configuration values, not a live spend ledger, and can change intraday. Combining them with `get_ads_perf`'s today's-`Spend` doesn't produce a reliable real-time remaining-budget figure either, since that metric is subject to the T+2 processing delay (see PLATFORM_NOTES.md). Tell the user this can't be computed reliably rather than presenting a subtraction as if it were precise.

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
- **`aiActionSettings`** (object): action-space on/off switches, returned as a
  nested structure with 5 sub-modules: `bidOptimization`, `structOptimization`,
  `budgetOptimization`, `targetOptimization`, `brandOptimization`. Each sub-module
  contains `xxxStatus` fields where `0`=off (action space disabled), `1`=on (active).
  When on, the mode (AI vs Rule) is determined by the corresponding
  `aiAutomation` entry (see below). Read-side field names may differ from write-side
  field names (e.g. read `budgetDaypartStatus` vs write `budgetDaypartActionStatus`).
- **`aiAutomation`** (object): rule mode switches, returned as a map keyed by rule
  number (e.g. `"13"`, `"19"`). Each entry has `status` (`0`=AI mode, `1`=Rule mode),
  plus Rule template contents (`condition`, `excuteDays`, etc.) when in Rule mode.
  An absent key means no Rule template exists for that action space. To determine
  the effective mode of an action space on read: check both
  `aiActionSettings.xxxStatus` (is it on?) and the corresponding `aiAutomation`
  entry's `status` (AI=0 or Rule=1?). The `noRule` capabilities
  (`budgetRedistribute`, `bidAmazonBusiness`) have no `aiAutomation` entry — only on/off.

**Filterable fields**:

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

**Return fields**: `amazonCampaignId` (long), `enabledRuleTypes` (array[int] — enabled rule type codes), `enabledRuleNames` (array[string] — human-readable rule names, already localized; there is no separate `Text` companion field for this one since the values are already human-readable)

**ruleType enum** (meaning of the ints in `enabledRuleTypes`):

| ruleType | Name | Description |
|---|---|---|
| 2 | Dayparting | Bid dayparting |
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
