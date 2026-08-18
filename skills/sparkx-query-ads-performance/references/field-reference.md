# Field & Metrics Reference — `get_ads_perf`

## Supported Dimension Fields (by entity)

### Campaign

| Field | Description | Enum |
|---|---|---|
| `campaign.campaignId_` | Campaign ID | — |
| `campaign.campaignName_` | Campaign name | — |
| `campaign.campaignType_` | Campaign type | `sponsoredProducts` / `sponsoredBrands` / `sponsoredDisplay` |
| `campaign.campaignState_` | State | `enabled` / `paused` / `archived` |
| `campaign.campaignServingStatus_` | Serving status | — |
| `campaign.biddingStrategy_` | Bidding strategy | `legacyForSales` / `autoForSales` / `manual` / `ruleBased` |
| `campaign.targetingType_` | Targeting mode | `manual` / `auto` |
| `campaign.costType_` | Cost type | `cpc` / `vcpm` / `fixedprice` |
| `campaign.isAiCreate_` | Whether AI-created | `0` (no) / `1` (yes) |
| `campaign.dailyBudget_` | Daily budget | — |
| `campaign.currentBudget_` | Current budget | — |

**⚠️ "How much budget do I have left today?" cannot be reliably answered by combining these fields with today's `Spend`.** `dailyBudget`/`currentBudget` are configuration values (from `get_entity_metadata`, or these dimension fields here) that can change intraday and aren't a live spend-tracking ledger; today's `Spend` from `get_ads_perf` is also subject to the T+2 data delay, so "today's spend so far" is frequently incomplete, not a live real-time figure. Subtracting one from the other (`remaining = dailyBudget − today's Spend`) produces a number that looks precise but is not reliable. If asked this, say plainly that real-time remaining budget can't be computed reliably from this data — don't present a computed number with false confidence. You can still show the configured `dailyBudget` and the most recent *complete* day's `Spend` as background context, clearly labeled as such (not as "today's remaining budget").

| Field | Description | Enum |
|---|---|---|
| `campaign.campaignStartDate_` / `campaign.campaignEndDate_` | Start/end date | — |
| `campaign.portfolioId_` | Portfolio ID | — |
| `campaign.aiGroupId_` | AI managed group ID | — |
| `campaign.campaignAiFirstOnDate_` / `campaign.campaignAiLastOnDate_` / `campaign.campaignAiLastOffDate_` / `campaign.campaignAiStudyDate_` | AI managed lifecycle dates | — |
| `campaign.offAmazonBudgetControlStrategy_` | Off-Amazon budget strategy | `MINIMIZE_SPEND` / `MAXIMIZE_REACH` |
| `campaign.siteRestrictions_` | Site restrictions | `AMAZON_BUSINESS` / `AMAZON_HAUL` |
| `campaign.bidAdjustmentTopOfSearch_` | Top-of-search bid adjustment % | — |
| `campaign.bidAdjustmentProductPage_` | Product-page bid adjustment % | — |
| `campaign.bidAdjustmentRestOfSearch_` | Rest-of-search bid adjustment % | — |

### AdGroup

| Field | Description | Enum |
|---|---|---|
| `adGroup.adGroupId_` | Ad group ID | — |
| `adGroup.campaignId_` | Parent campaign ID | — |
| `adGroup.adGroupName_` | Ad group name | — |
| `adGroup.adGroupState_` | State | `enabled` / `paused` / `archived` |
| `adGroup.adGroupServingStatus_` | Serving status | — |
| `adGroup.defaultBid_` | Default bid | — |
| `adGroup.sdBidOptimization_` | SD bid optimization | `conversions` / `clicks` / `reach` |

### Target

| Field | Description | Enum |
|---|---|---|
| `target.targetId_` | Target ID | — |
| `target.adGroupId_` / `target.campaignId_` | Parent IDs | — |
| `target.targetText_` | Target text (keyword/ASIN/category) | — |
| `target.targetMatchType_` | Match type | `broad` / `phrase` / `exact` / `asinSameAs` / `asinCategorySameAs` / `asinAccessoryRelated` / `asinSubstituteRelated` / `asinExpandedFrom` / `keywordGroupSameAs` / `queryHighRelMatches` / `queryBroadRelMatches` |
| `target.targetState_` | State | `enabled` / `paused` / `archived` |
| `target.targetServingStatus_` | Serving status | — |
| `target.targetBid_` | Set bid | — |

**⚠️ No negative-targeting variant exists in `targetMatchType_`'s enum above.** There is no documented way to query the current list of negative keywords/ASINs/brands via this tool or `get_entity_metadata`. `get_operation_log` only tells you *when* a negative target was added/removed (`targetTypes`: `negativeKeyword`/`negativeAsin`/`negativeBrand`), not a queryable current snapshot. Treat "show me my negative keywords" as a genuine tool capability gap, not something to route around with an undocumented filter.
| `target.targetCurrentBid_` | Live current bid (real-time value, may differ from `target.targetBid_` if adjusted by rules/AI) | — |
| `target.targetPreCurrentBid_` | Previous day's bid | — |
| `target.targetingType_` | Targeting category | `keyword` / `target` / `auto` / `audience` |

### SearchTerm (no entity prefix on fields — see naming rules above)

| Field | Description | Enum |
|---|---|---|
| `query_` | Search term text | — |
| `matchType_` | Match type | `broad` / `phrase` / `exact` / `targeting` / `substitutes` / `complements` / `loose-match` / `close-match` / `asin-expanded` / `asin` / `category` |
| `targetId_` | Related target ID | — |
| `adGroupId_` / `campaignId_` | Parent IDs | — |

### Placement

| Field | Description | Enum |
|---|---|---|
| `placement.campaignId_` | Campaign ID | — |
| `placement.placement_` | Placement | `topOfSearch` / `productPage` / `restOfSearch` |
| `placement.multiplier_` | Bid adjustment % | — |

### ProductAd

| Field | Description | Enum |
|---|---|---|
| `productAd.amazonAdId_` | Amazon ad ID | — |
| `productAd.adGroupId_` | Parent ad group ID | — |
| `productAd.asin_` | ASIN | — |
| `productAd.sku_` | SKU | — |
| `productAd.productAdState_` | State | `enabled` / `paused` / `archived` |

### ASIN (child)

| Field | Description |
|---|---|
| `asin.asin_` | Child ASIN |
| `asin.parent_asin_` | Parent ASIN |
| `asin.sku_` | SKU |
| `asin.asinTitle_` | Product title |
| `asin.asinBrand_` | Brand |
| `asin.asinOpenDate_` | First-live date |
| `asin.asinCategoryInfo_` | Category info |
| `asin.asinBsr_` | Best Seller Rank |
| `asin.asinPrice_` | Price — **always local currency, even in a multi-profile USD-normalized query** |
| `asin.asinFbaQunatity_` | FBA inventory quantity |
| `asin.asinInventoryStatus_` | `Active` / `Inactive` / `Incomplete` |
| `asin.asinSpEligibilityStatus_` / `asin.asinSbEligibilityStatus_` / `asin.asinSdEligibilityStatus_` | Ad-type eligibility, `ELIGIBLE` / `INELIGIBLE` |
| `asin.asinIsDelete_` | `0` (active) / `1` (deleted) |

### ParentAsin

| Field | Description |
|---|---|
| `parentAsin.parent_asin_` | Parent ASIN |
| `parentAsin.parentAsinTitle_` / `parentAsin.parentAsinBrand_` / `parentAsin.parentAsinOpenDate_` / `parentAsin.parentAsinCategoryInfo_` / `parentAsin.parentAsinBsr_` / `parentAsin.parentAsinPrice_` / `parentAsin.parentAsinFbaQunatity_` / `parentAsin.parentAsinInventoryStatus_` / `parentAsin.parentAsinIsDelete_` | Same semantics as ASIN, rolled up to parent |

### ProductLine

| Field | Description |
|---|---|
| `productLine.productLineParentId_` / `productLine.productLineParentName_` | Top-level product line |
| `productLine.productLineChildId_` (0 = no sub-tag) / `productLine.productLineChildName_` | Sub product line |
| `productLine.productLineCreator_` | Creator |
| `productLine.asin_` | Associated ASIN |

### Portfolio

| Field | Description | Enum |
|---|---|---|
| `portfolio.portfolioId_` | Portfolio ID | — |
| `portfolio.portfolioName_` | Portfolio name | — |
| `portfolio.portfolioState_` | State | `enabled` / `paused` / `archived` |
| `portfolio.portfolioServingStatus_` | Serving status | — |
| `portfolio.portfolioStartDate_` / `portfolio.portfolioEndDate_` | Date range | — |
| `portfolio.portfolioBudget_` | Budget amount | — |
| `portfolio.portfolioBudgetType_` | Budget type | `dateRange` / `monthlyRecurring` |
| `portfolio.portfolioCreator_` | Creator | — |

### AI Group

| Field | Description | Enum |
|---|---|---|
| `aiGroup.aiGroupId_` / `aiGroup.aiGroupName_` | AI managed group ID/name | — |
| `aiGroup.aiStatus_` | Status | `0` (not started) / `1` (running) / `2` (paused) |
| `aiGroup.targetAcos_` | Target ACOS (%) | — |
| `aiGroup.aiTargetType_` | Objective | `1` (Drive Growth) / `2` (Maintain Stable Orders) / `3` (Event Boost) |
| `aiGroup.aiPersonality_` | AI aggressiveness | `1` (very conservative) … `5` (very aggressive) |
| `aiGroup.aiGroupStatusOnDate_` / `aiGroup.aiGroupLastStatusOnDate_` / `aiGroup.aiGroupLastStatusOffDate_` | Lifecycle dates | — |
| `aiGroup.aiCreator_` / `aiGroup.aiGroupCreateTime_` | Creator / create time | — |
| `aiGroup.budgetDynamicStatus_`, `aiGroup.competitorStatus_`, `aiGroup.negativeTargetStatus_`, `aiGroup.targetPausedAddStatus_`, `aiGroup.bidOptimizationStatus_`, `aiGroup.bidDaypartStatus_`, `aiGroup.bidPerformanceStatus_`, `aiGroup.bidAdPlaceStatus_`, `aiGroup.structOptimizationStatus_`, `aiGroup.structPauseProductStatus_`, `aiGroup.structPauseCampaignStatus_`, `aiGroup.budgetDaypartStatus_`, `aiGroup.budgetRedistributeStatus_` | Action Space feature switches (`0`/`1`) | — |

### AutomationRule

| Field | Description |
|---|---|
| `automationRule.campaignId_` | Campaign ID |
| `automationRule.ruleDaypartingBid_` | Dayparting bid rule enabled (`0`/`1`) |
| `automationRule.ruleAddSearchTerm_` | Harvest search term rule |
| `automationRule.ruleAddNegativeKeyword_` | Harvest negative keyword rule |
| `automationRule.rulePauseCampaign_` / `automationRule.rulePauseCampaignV2_` | Auto-pause campaign rule |
| `automationRule.ruleDaypartingBudget_` | Dayparting budget rule |
| `automationRule.ruleTargeting_` | Auto-pause targeting rule |
| `automationRule.ruleInventory_` | Inventory rule |
| `automationRule.ruleBudget_` | Performance-based budget rule |
| `automationRule.rulePlacement_` | Placement adjustment rule |

### Profile

| Field | Description | Enum |
|---|---|---|
| `profile.profileId_` | Profile ID | — |
| `profile.profileName_` | Profile name | — |
| `profile.countryCode_` | Country | `US`/`CA`/`MX`/`JP`/`UK`/`DE`/`FR`/`IT`/`ES`/`NL`/`AU`/`SG`/`BR`/`SE`/`AE`/`PL`/`IN`/`TR`/`SA`/`BE`/`EG`/`ZA` |
| `profile.profileDailyBudgetCap_` | Account daily budget cap | — |
| `profile.profileUseBudgetCap_` | Whether budget cap enforced | — |

**Whenever `profileIds` has more than one entry, add `select: ["profile.profileId_", "profile.profileName_", ...]`** (alongside whatever other entity you're grouping by) so each row can be attributed to a specific store.

## Metrics × Entity Support Matrix

**Only request metrics valid for the chosen `factEntity`.** Requesting an unsupported combination (e.g. `NTBOrders` on `placement`, or `TACOS` on anything but `asin`) is a common source of invalid-call errors.

| Metric category | campaign | adGroup | target | searchTerm | placement | productAd | asin |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Basic (Impressions/Clicks/Spend/AISpend) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Sales/Conversion (Sales/Conversions/Units + Same/Other SKU) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Derived (CTR/CVR/CPC/CPA/ACOS/ROAS) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| NTB (NTBOrders/NTBSales/NTBUnits etc) | ✓ | ✓ | ✓ | ✓ | ✗ | ✓ | ✗ |
| DPV (DPV/DPVClick/DPVR) | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✗ |
| Attribution (OrdersClick/SalesvCPM etc) | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ | ✗ |
| Viewability (ViewableImpressions/vCPM) | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Video (VideoCompleteViews/VTR etc) | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Impression share (TopOfSearchIS) | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| AI (AISales/AIACOS/AIROAS) | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| ASIN business (TotalSalesAmount/TACOS/Sessions etc) | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

## Metrics Reference

### Basic metrics
`Impressions`, `Clicks`, `Spend`, `AISpend`

### Conversion metrics
`Sales`, `Conversions` (maps from user's "Orders" — there is no metric literally named `Orders`), `Units`, `SalesSameSKU`, `ConversionsSameSKU`, `UnitsSameSKU`, `SalesOtherSKU`, `ConversionsOtherSKU`, `UnitsOtherSKU`

### Derived ratio metrics
| Metric | Formula |
|---|---|
| CTR | Clicks/Impressions×100 |
| CVR | Conversions/Clicks×100 |
| CPC | Spend/Clicks |
| CPA | Spend/Conversions |
| ACOS | Spend/Sales×100 |
| ROAS | Sales/Spend |

**⚠️ `CTR`/`CVR`/`ACOS` are confirmed returned pre-scaled ×100** — a value of `17.61` means 17.61%. Don't re-scale the number, but **append `%`** when presenting it to the user: "ACOS is 17.61%". Filters stay on the raw ×100 scale with no `%` in the JSON (`{"ACOS": {"<": 20}}`). `TACOS` and other `*Rate`/`*Percentage`-named metrics are **not independently confirmed** to be on this scale — relay their raw value as-is with no `%` until backend confirms. `CPC`/`CPA`/`ROAS` are plain ratios, not percentages at all.

### NTB (New-to-Brand) metrics
`NTBOrders`, `NTBUnits`, `NTBSales`, `NTBOrdersRate`, `NTBUnitsRate`, `NTBSalesRate`

⚠️ **Not available for SP (sponsoredProducts)** — must filter `campaignType` to SB/SD. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md) for split-query rules.

### DPV (Detail Page View) metrics
`DPV`, `DPVClick`, `DPVvCPM`, `DPVR`

⚠️ **Not available for SP** — SB/SD only. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### Attribution-window metrics
`OrdersClick`, `UnitsClick`, `SalesClick`, `OrdersvCPM`, `UnitsvCPM`, `SalesvCPM`

⚠️ **Not available for SP** — SB/SD only; not supported on placement or asin entity. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### Viewability / vCPM metrics
`ViewableImpressions`, `ViewImpressions`, `ViewableImpressionsRate`, `vCPM`

⚠️ `ViewImpressions` is **SD vCPM only**; `ViewableImpressions`/`ViewableImpressionsRate` available for all types. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### Video metrics
`VideoUnmutes`, `VideoFirstQuartileViews`, `VideoMidpointViews`, `VideoThirdQuartileViews`, `VideoCompleteViews`, `Video5SecondViews`, `Video5SecondViewRate`, `VTR`, `vCTR`

⚠️ **Not available for SP** — SB/SD only. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### Impression share
`TopOfSearchIS`

⚠️ **Only supported on campaign and target `factEntity`** — not available on other entities. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### AI metrics (campaign entity only)
`AISales`, `AIACOS`, `AIROAS`

⚠️ **Only supported on campaign `factEntity`**. See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### ASIN business metrics (asin entity only)
`TotalSalesAmount`, `OrderCount`, `UnitCount`, `AverageOrderPrice`, `AverageProductPrice`, `TACOS`, `CPO`, `Sessions`, `UnitSessionPercentage`, `PageViews`, `GlanceViews`, `BuyBoxPercentage`, `OrderedUnits`, `OrderedRevenue`, `ShippedUnits`, `ShippedRevenue`, `ShippedCogs`, `CustomerReturns`, `NetPPM`, `UnavailabilityRate`, `AdsSalesRate`, `AdsOrdersRate`, `AdsUnitsRate`, `AdsSalesSameSKURate`, `AdsOrdersSameSKURate`, `AdsCVR`, `OrganicSales`, `OrganicOrders`, `ShippedAverageProductPrice`, `ShippedTACOS`, `OrderedAverageProductPrice`, `OrderedTACOS`

⚠️ **Only supported on asin `factEntity`** — do NOT join with Campaign/AdGroup/aiGroup dimensions (causes row duplication). See [`ad-type-dependent-metrics.md`](ad-type-dependent-metrics.md).

### Common user-to-field mapping
| User says | Standard field |
|-----------|---------------|
| Impressions | Impressions |
| Clicks | Clicks |
| Spend | Spend |
| Sales | Sales |
| Orders | Conversions |
| Units | Units |
| CTR | CTR |
| CVR | CVR |
| ACOS | ACOS |
| ROAS | ROAS |

---

**Enum i18n**: When presenting enum values to the user or translating between API values and display labels, consult [`enum-i18n.md`](enum-i18n.md) for the complete ZH/EN/JA mapping of all enum fields documented above.
