# Xnurta MCP Read Tools — Shared Platform Notes

This file holds behavior that is **common to all 3 read tools** (`get_ads_perf`, `get_entity_metadata`, `get_operation_log`). It ships as `references/platform-notes.md` **inside this skill's own folder** — this copy travels with the `query-ads-performance` skill even if a Skill Hub installs/updates/downloads one skill folder at a time, since the reference file is nested under the same folder, not a sibling directory. The other two Xnurta skills (`query-entity-metadata`, `query-operation-log`) each carry an identical copy of this file under their own `references/` folder for the same reason — this is intentional duplication for delivery robustness, not a mistake. Read this once per session before making tool-selection decisions.

## Auth Flow

All 3 tools authenticate via Bearer Token, handled by a shared pipeline (pre-ratelimit → auth → business ratelimit → error mapping). Each tool method itself only does param validation + business logic.

**Before calling any of the 3 read tools, you must first call `get_user_authorized_context`** to obtain the current token's authorized `profileIds`, `tenantId`, `userId`.

`get_user_authorized_context` response:
```json
{
  "isError": false,
  "toolName": "get_user_authorized_context",
  "userId": "12345",
  "tenantId": "2064",
  "profiles": [
    {"profileId": 4404871489220462, "profileName": "Star-Store-US", "countryCode": "US"},
    {"profileId": 9283746501928374, "profileName": "Star-Store-JP", "countryCode": "JP"}
  ]
}
```

How to use this:
- `profiles[].profileId` → the `profileIds` param for all 3 tools
- `profiles[].countryCode` → resolve "US store" style user references to a profileId
- `profiles[].profileName` → resolve store-name references
- **User didn't specify a store** → default to passing **all** authorized `profileIds` (cross-store aggregate; monetary metrics come back in USD — see Currency below)

## Permission Scopes

Each tool requires a specific OAuth scope on the token. If a call fails with an auth/permission error, this is the likely cause — tell the user their token/role is missing the corresponding scope rather than guessing at a query bug.

| Tool | Required scope |
|---|---|
| `get_ads_perf` | `amazon_sa:performance:read` |
| `get_entity_metadata` | `amazon_sa:ads_configuration:read` |
| `get_operation_log` | `amazon_sa:ads_logs:read` |

## profileIds Intersection Logic

**All 3 tools intersect the requested `profileIds` with the token's authorized `profileIds`** — they do NOT reject unauthorized IDs outright:

- effective query scope = requested `profileIds` ∩ authorized `profileIds`
- if the intersection is **empty**, the tool returns an **empty result set**, not an error
- the response's `effectiveProfileIds` field reflects the actual (post-intersection) scope — always check this field, especially when the user asked about a specific store, to confirm the store you meant was actually in scope

This matters for "replace profile X with all currently-authorized profiles" style requests: passing a profile ID not in scope silently drops it rather than erroring, so cross-check `effectiveProfileIds` against what you expected before reporting a "0 results" answer as if it were real business data.

## Diagnosing Zero/Empty Results

Any of the 3 tools can legitimately return zero rows for reasons that have nothing to do with "no data exists." Before telling the user "there's no data for that," work through this sequence — don't jump straight to that conclusion from the first empty response:

1. **Check `isError`.** If `true`, this isn't an empty-data situation at all — it's an error (see the error-handling section above). Handle it as an error, not as "no results."
2. **Check `effectiveProfileIds`.** If it's empty or missing profiles you expected, the requested `profileIds` didn't actually intersect with the token's authorized set (see profileIds Intersection Logic above) — the query never ran against the store(s) you meant. Fix the profile scope and retry before concluding anything about the data.
3. **Check the date range for T+2 delay effects (`get_ads_perf` only) — but note this is different from a range violation.** If `dateStart`/`dateEnd` genuinely exceed the 90-day span or 15-month lookback, that's a request-construction mistake that should surface as `isError:true` (`errorType: invalid_params`) per the error-handling rules above — it should not come back as a silent empty success. If you're instead seeing `isError:false` with empty/incomplete rows for a *valid* range that includes "today" or very recent days, that's the T+2 delay: recent data genuinely hasn't finished processing yet, which is a legitimate (if unhelpful) empty-but-correct result — not a bug and not "no data exists," just "not processed yet."
4. **Confirm the requested fields/metrics are actually valid for the `factEntity`/`entity` in use** (see the Metrics × Entity Support Matrix). Requesting an unsupported combination is documented as a common source of `invalid_params` errors — so this should also come back as `isError:true`, not a silent empty result. If you're getting empty-but-non-error rows, an invalid metric/entity combination is an unlikely explanation; look at filters and data existence instead (steps 5-6).
5. **Remove your business filters (`filters`) and retry.** If removing filters (state=enabled, spend thresholds, etc.) suddenly returns rows, the entity/data exists but didn't match your filter conditions — the honest answer to the user is "nothing matches your filter criteria," not "you have no data at all." These are different statements — don't conflate them.
6. **If you still get zero rows after 1-5, confirm the entity itself exists** via `get_entity_metadata` (e.g. does this campaign ID actually exist for this profile?). A nonexistent entity and an existing-but-inactive-in-this-window entity warrant different answers to the user.

**Only after working through this sequence** can you responsibly tell the user "there's genuinely no data/activity in that range" — and even then, be specific about what you checked (e.g. "no spend recorded for campaign X between these dates, and the campaign does exist and is enabled") rather than a bare "no data found."

## Error Response Format

All 3 tools use the same error envelope:
```json
{
  "isError": true,
  "errorType": "invalid_params",
  "message": "Parameter 'profileIds' is required and cannot be empty"
}
```

**`errorType` enum**:

| errorType | Meaning | Common cause |
|---|---|---|
| `invalid_params` | Bad parameters | Missing required param, bad date format, date span over limit |
| `rate_limited` | Rate-limited | Business-level rate limit triggered — retry after a delay |
| `query_error` | Query failure | Backend service call failed |
| `serialization_error` | Serialization failure | Rare fallback case |

**Rate-limit errors carry extra fields**:
```json
{
  "isError": true,
  "errorType": "rate_limited",
  "dimension": "tenant_per_minute",
  "retryAfterSeconds": 30,
  "message": "Rate limit exceeded on dimension [tenant_per_minute]. Retry after 30 seconds."
}
```

**How to handle each errorType when talking to the user:**
- `invalid_params` → this is almost always a query-construction bug on the agent's side (bad field name, missing required param, date range too wide). Fix the call and retry — don't tell the user "the tool doesn't support this" until you've checked field names/param names against this skill's reference tables.
- `rate_limited` → wait `retryAfterSeconds` before retrying the *same* call. Don't immediately retry in a loop. If it recurs, tell the user the query is being throttled and suggest narrowing scope (smaller date range, fewer profiles) to reduce call volume.
- `query_error` → backend failure, not a param problem. Retry once; if it persists, tell the user the data source is temporarily unavailable rather than implying their question is unanswerable.
- `serialization_error` → rare edge case; report to the user as a transient tool error, retry once.

Always check `isError` before reading `rows` — do not assume a response without `isError:false` explicitly checked is safe to parse as data.

## Pagination Rules

| Tool | Pagination style | Default pageSize | Max pageSize |
|---|---|---|---|
| `get_ads_perf` | `page` + `pageSize` | 100 | 500 |
| `get_entity_metadata` | `page` + `pageSize` | 100 | 500 |
| `get_operation_log` | **limit-only, no `page` param** | 50 | 200 |

`get_operation_log` always returns the most recent N rows time-descending; there is no way to page past the first N. **Default to `pageSize: 200`** and always check `truncated`. If `truncated=true`, split the date range into non-overlapping sub-windows and recurse (see `query-operation-log`'s "Getting a Complete Count" section) — this is the primary remedy. Narrowing by `entities`/`resourceIds`/`operationType`/`changeBy` instead of splitting changes *what* you're searching for, not just how much of it you retrieve, so only use it as a last resort (single day still truncated, or the user explicitly wants one type of change only) — and flag the result as partial when you do.

## Date Range Limits

**⚠️ Illustrative dates**: any concrete date shown in this skill's own example files (e.g. `2026-06-01`) is for format illustration only — it will eventually fall outside the 15-month lookback window as time passes. Always substitute real dates that satisfy the constraints below at call time; never copy an example's literal date value into a live call without checking it's still within range.

| Tool | Max date span | Earliest lookback | Format |
|---|---|---|---|
| `get_ads_perf` | 90 days | 15 months | `YYYY-MM-DD` |
| `get_operation_log` | 90 days | 15 months | `YYYY-MM-DD` |
| `get_entity_metadata` | No date limit (no date params at all) | — | — |

These are two **separate** hard constraints on the `dateStart`/`dateEnd` request parameters, not "auto-truncate":
1. **Max span is 90 days, counting both endpoints (inclusive).** Precisely: `dateEnd - dateStart` (as a plain date subtraction) must be **≤ 89**, not ≤ 90 — since both `dateStart` and `dateEnd` count as covered days, a subtraction of 90 actually spans 91 inclusive calendar days, one more than the limit. Think of it as "≤ 90 inclusive calendar days," and derive the subtraction bound (89) from that, not the other way around. **Weekly/monthly aggregation (`select: [..., "toMonday(...) as week"]`) does NOT let you bypass the span limit either way** — it only reduces the number of *rows* a single already-compliant call returns. If the user wants a window longer than 90 days (e.g. "performance over the last 12 months by month"), you **must first split into multiple calls, each spanning ≤90 inclusive days** (a full year typically needs **5** such calls, not 4 — calendar quarters are 91-92 days and don't reliably fit under the 90-day cap, so don't split by quarter; see query-ads-performance's aggregation-over-time example for a worked, verified split), and *then*, within each call, optionally use week/month aggregation in `select` to reduce that call's row count. Splitting is mandatory; aggregation is optional and orthogonal.
2. `dateStart` cannot be more than 15 months before today — going further back returns an error, it does not silently clamp the range for you.

If a user's requested range exceeds 90 days, **proactively split it** into sequential ≤90-day calls rather than sending one oversized request and waiting for it to fail.

**Data delay**: ad performance data has a T+2 delay. "Today"'s data is usually incomplete. When the user doesn't specify an end date, default `dateEnd` to **yesterday**, not today.

### Date Format: Query Params vs. Data Fields

**These are two different formats — do not use one where the other is expected.**

| Where | Format | Example |
|---|---|---|
| `dateStart`/`dateEnd` request params (`get_ads_perf`, `get_operation_log`) | `YYYY-MM-DD` | `"2026-06-01"` |
| `date` dimension field returned in `get_ads_perf` rows (daily grouping) | `YYYYMMDD` (no dashes) | `"20240601"` |
| `campaignStartDate`/`campaignEndDate` fields in `get_entity_metadata` (both as returned values and as filter values) | `YYYYMMDD` (Ymd, no dashes) | `"20260101"` |
| `createdDate` field in `get_operation_log` rows | Full timestamp, UTC | `"2026-06-15 14:30:00"` |

Concretely: when you *request* a date range, always use `YYYY-MM-DD`. When you *read or filter* the `date`/`campaignStartDate`/`campaignEndDate` data fields, use `YYYYMMDD` with no separators — e.g. `{"campaignStartDate": {">=": "20260101", "<=": "20260131"}}`, not `{"campaignStartDate": {">=": "2026-01-01"}}`. Getting this backwards is a common cause of a filter silently matching nothing or an `invalid_params` error.

### ⚠️ Unconfirmed: What timezone does `dateStart`/`dateEnd` filtering use?

**This is a genuine gap in the platform spec, not something to guess at.** The spec documents inconsistent timezone conventions on different fields — e.g. `get_operation_log`'s `createdDate` and `aiGroup.aiPersonalityUpdatedAt` are UTC, while `aiGroup.createTime` is explicitly "store timezone" (profile-local) — but it never states which timezone `dateStart`/`dateEnd` are interpreted in when resolving a date boundary like "today" or "yesterday" for filtering.

This matters most for:
- **Multi-country `profileIds` queries**: if you resolve "yesterday" using the *caller's* timezone (e.g. Beijing time) while the underlying filter is actually applied in the *store's* timezone (e.g. US Pacific) or in UTC, a query for "US store, yesterday's operations" can be off by up to a full day near the boundary — pulling the wrong day's data, or missing part of the intended day.
- **`get_operation_log`** specifically, since its `createdDate` values are UTC but the request-level `dateStart`/`dateEnd` boundary this filters against is not documented as being interpreted in UTC (or any other specific zone).

**Do not assume or assert a specific answer** (e.g. "it's UTC" or "it's store-local") — this needs explicit confirmation from the team building the new tool before this skill can responsibly resolve relative dates in multi-timezone scenarios. Until confirmed:
- When precision matters (the user is asking about a boundary-sensitive window like "yesterday" for a specific non-UTC store), say you're not certain which timezone the boundary uses and that results near day boundaries should be treated as approximate.
- Prefer giving the user a same-timezone reference point when possible (e.g. "results for 2026-07-19" rather than just "yesterday") so any boundary ambiguity is visible rather than silently baked into an unlabeled relative date.

## Currency Rules

Amount fields carry a `currency` indicator, but the exact mechanism differs by tool/entity — read carefully, this is a common source of dollar-amount misinterpretation with multi-store customers.

### get_ads_perf

| Scenario | Outer `currency` | Behavior |
|---|---|---|
| Single profile | That profile's local currency code (e.g. `"JPY"`) | All monetary metrics (Spend/Sales/CPC/etc) in local currency |
| Multiple profiles | `"USD"` | All monetary metrics auto-converted to USD |

**Exception**: the `asin.asinPrice_` dimension field (product list price) is **always local currency** and is NOT affected by the USD conversion above, even in a multi-profile query. If `select` includes `asin.asinPrice_` on a multi-profile call, that one field's value does not match the outer `currency=USD` — cross-reference `profile.countryCode_` to determine its actual currency.

### get_entity_metadata

| Entity | Scenario | Outer `currency` | Per-row `currency` field | Notes |
|---|---|---|---|---|
| campaign/adGroup/portfolio/etc | Single profile | Local currency code | none | `dailyBudget` etc in local currency |
| campaign/adGroup/portfolio/etc | Multi profile | `"USD"` | none | Amount fields pre-converted to USD by backend |
| **asin** | Single profile | Local currency code | none | `asinPrice`/`parentAsinPrice` in local currency |
| **asin** | Multi profile | **not present** | **each row carries a `currency` field** | Product pricing is not FX-converted; each row's currency is identified individually |

`asin` multi-profile example:
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

**Rule of thumb**: outer `currency` present → all rows share that currency. Outer `currency` absent → check each row's own `currency` field.

### get_operation_log

Amount-bearing fields (e.g. `previousValue`/`newValue` for a `dailyBudget` change) are always **local currency** — there is no cross-profile USD conversion for logs. Each row carries a `currencyCode` field (e.g. `"USD"`/`"JPY"`) mapped from that row's `countryCode`. There is no outer `currency` field for this tool.

## Ratio Metric Display Rule

**Two tiers — do not treat every ratio-shaped metric the same way.** Only some fields have confirmed evidence of a ×100/percentage scale in the platform spec; the rest are unconfirmed and must be handled more conservatively.

### Tier 1 — Confirmed ×100/percentage scale

`ACOS` (`Spend/Sales×100`), `CTR` (`Clicks/Impressions×100`), `CVR` (`Conversions/Clicks×100`) — confirmed by their documented formulas and by the tool's own response example (`"ACOS": 17.61`, i.e. 17.61%). `aiGroup.targetAcos_` (`get_entity_metadata`) is confirmed by its explicit label "Target ACOS (percentage)". `UnitSessionPercentage` is confirmed the same way (explicitly labeled "percentage" in the spec). For these fields:

- The number is already ×100-scaled — do not multiply or divide by 100 again.
- **Append a `%` sign when presenting the value to the user**: if the tool returns `"ACOS": 17.61`, say "ACOS is 17.61%". If `targetAcos` returns `35`, say "target ACOS is 35%".
- When constructing a filter, pass the threshold on this same ×100 scale, as a plain number with **no `%` in the JSON**: `{"ACOS": {"<": 20}}` for "ACOS under 20%" — not `{"ACOS": {"<": 0.2}}` (which means "under 0.2%", almost never intended).

### Tier 2 — Unconfirmed scale (needs backend/product confirmation)

`TACOS`, `ShippedTACOS`, `OrderedTACOS`, `NTBOrdersRate`, `NTBUnitsRate`, `NTBSalesRate`, `ViewableImpressionsRate`, `AdsSalesRate`, `AdsOrdersRate`, `AdsUnitsRate`, `AdsSalesSameSKURate`, `AdsOrdersSameSKURate`, `Video5SecondViewRate` — the platform spec gives only a metric name for these, with **no formula and no example value**, so their scale is not independently verified the way Tier 1 is.

**`AdsCVR` is a special case, not just "unconfirmed"**: its documented formula is `Conversions/Clicks` — explicitly **without** `×100`, unlike `CVR`'s formula which explicitly includes `×100`. This is active evidence that `AdsCVR` may be on a *different* scale (a plain 0–1 ratio) than `CVR`, not merely an assumption gap. Do not treat it as interchangeable with `CVR`.

For all Tier 2 fields:
- Relay the tool's raw value exactly as returned — do not assume it's a percentage, do not multiply/divide by 100, and do not append a `%` sign, since you don't actually know if the number is already ×100-scaled or a 0–1 ratio.
- Do not construct filters against these fields using an assumed percentage scale (e.g. don't guess that `20` means "20%") — if the user wants to filter on one of these, either ask what scale they mean or note the ambiguity, and flag to Xnurta backend/product that this field's scale needs explicit confirmation before the skill can give guidance as confident as Tier 1's.

`ROAS`, `CPC`, `CPA` are **not percentages at all** — they're plain ratios/currency-per-unit values (e.g. `ROAS: 5.68`), not covered by either tier.

## Multi-Profile Row Attribution

When a call spans **more than one** `profileId`, make sure the response can actually be attributed back to a specific store before presenting results — rows with the same campaign/entity name across different stores are otherwise indistinguishable to the user.

- **`get_ads_perf`**: add `"profile.profileId_"` (and usually `"profile.profileName_"`) to `select` whenever `profileIds` has more than one entry, so each row carries its store identity.
- **`get_entity_metadata`**: check whether the entity's returned rows include a `profileId` field (e.g. the `asin` entity does — see its Currency example). If the entity type you're querying doesn't surface `profileId` on its own, either issue one call per `profileId` or cross-reference by combining with a `get_ads_perf`/`asin` call that does carry `profileId`, before presenting a merged answer across stores.
- **`get_operation_log`**: every row already carries `profileId` (see ChangeLogVO fields) — map it to the store's `profileName` using the `profiles` list from `get_user_authorized_context` before describing a change to the user, especially when `entities`/`resourceIds` span multiple stores with same-named campaigns.

## Tool Selection Decision Tree

```
User intent:
├── Has a time range + wants metric data (spend/sales/ACOS/clicks/...)
│   └── → get_ads_perf
├── Asking about config/list/status (no metrics, no date range)
│   ├── "what campaigns exist" → get_entity_metadata (entity=campaign)
│   ├── "AI managed group config" → get_entity_metadata (entity=aiGroup)
│   ├── "product info / ASIN" → get_entity_metadata (entity=asin)
│   ├── "portfolio list" → get_entity_metadata (entity=portfolio)
│   └── "which automation rules are enabled on this campaign" → get_entity_metadata (entity=automationRule)
├── "Who did what" / "what did AI change" / "change history"
│   └── → get_operation_log
└── Needs both metrics AND config
    └── get_ads_perf first for the data → get_entity_metadata to enrich with names/config
```

**Common combined-call patterns**:
- "What's the bidding strategy of the highest-spend campaign?" → `get_ads_perf` (find the top campaign) → `get_entity_metadata` (look up its config)
- "How did the AI managed group perform last week?" → `get_ads_perf` with `select` including `aiGroup.aiGroupName_`
- "What bids did AI change yesterday?" → `get_operation_log` with `changeBy=ai`, `actionType=Bid Increased/Bid Decreased`
- "Why did my ACOS go up?" (root-cause investigation, all 3 tools) → `get_ads_perf` twice to quantify the change and identify the affected campaign(s) → `get_operation_log` on that campaign over the same window to find candidate changes → `get_entity_metadata` for current config context. Report findings as correlation, not proven causation — see the dedicated ACOS root-cause example.

## Implicit Inference Rules

These are semantic mappings the agent should apply automatically when translating natural language into tool params.

### Metric name mapping (get_ads_perf)

| User says | Maps to metric | Note |
|---|---|---|
| Orders / conversions | `Conversions` | **Not** `Orders` — there is no metric literally named `Orders` |
| Spend / ad cost | `Spend` | — |
| Sales / revenue | `Sales` | — |
| Total sales | `TotalSalesAmount` | `asin` entity only |
| TACOS | `TACOS` | `asin` entity only |
| New-to-brand | `NTBOrders` + `NTBSales` | Pick based on what's asked |

### Status/filter inference (get_ads_perf, dimension fields use `entity.field_`; get_entity_metadata, plain camelCase — see each tool's own SKILL.md for the exact field format)

| User says | Inferred filter (ads_perf form) |
|---|---|
| AI-managed / AI ads | `{"aiGroup.aiStatus_": 1}` |
| Ads without AI enabled | `{"OR": [{"aiGroup.aiStatus_": 0}, {"aiGroup.aiStatus_": 2}]}` |
| Active / running ads | `{"campaign.campaignState_": "enabled"}` |
| Paused keywords | `{"target.targetState_": "paused"}` |
| "By portfolio" analysis | add `{"portfolio.portfolioId_": {">": 0}}` (exclude campaigns with no portfolio) |
| "By product line" analysis | add `{"productLine.productLineParentId_": {">": 0}}` |
| SP ads | `{"campaign.campaignType_": "sponsoredProducts"}` |
| SB ads | `{"campaign.campaignType_": "sponsoredBrands"}` |
| SD ads | `{"campaign.campaignType_": "sponsoredDisplay"}` |

### Time inference

**`get_ads_perf` is subject to the T+2 data delay** (see above) — default `dateEnd` to **yesterday** whenever the user doesn't give an explicit end date, since "today"'s ad performance data is usually incomplete.

| User says (get_ads_perf) | Inferred range |
|---|---|
| Last week | `dateStart` = last Monday, `dateEnd` = last Sunday |
| This month | `dateStart` = 1st of this month, `dateEnd` = yesterday |
| Last 30 days | `dateStart` = 30 days ago, `dateEnd` = yesterday |
| Quarter-to-date (QTD) | `dateStart` = 1st day of the current calendar quarter (Jan/Apr/Jul/Oct 1st), `dateEnd` = yesterday |
| Year-to-date (YTD) | `dateStart` = January 1st of the current year, `dateEnd` = yesterday |
| "Since this campaign launched" / "since we started" | **Not a fixed offset — look it up.** First query `get_entity_metadata` (`entity: "campaign"`) for that campaign's `campaignStartDate`, then use that as `dateStart`. **Constrained by the 15-month lookback**: if the launch date is more than 15 months ago, you cannot query the full lifecycle in one range — say so explicitly ("I can only go back 15 months; this campaign launched earlier than that, so this covers the most recent 15 months, not the full history") rather than silently truncating and presenting it as the complete picture |

**⚠️ Check for a reversed range before sending it — this month/QTD/YTD can produce `dateEnd < dateStart` on the first day of the period.** These rules all use "yesterday" as `dateEnd` (per the T+2 rule) but "1st of the period" as `dateStart`. If today is the 1st of the month/quarter/year, `dateStart` = today's date but `dateEnd` = yesterday — which is *before* `dateStart`, an invalid, reversed range. Concretely: querying "this month" on July 1st gives `dateStart=2026-07-01`, `dateEnd=2026-06-30` (reversed). The same happens for QTD on Jan/Apr/Jul/Oct 1st, and for YTD on January 1st. **Always compute `dateEnd` and compare it to `dateStart` before issuing the call.** If `dateEnd < dateStart`, do not send the request — the period genuinely has no completed days to report yet (e.g. "this month" on July 1st has zero elapsed complete days). Tell the user that directly ("this month just started, so there's no completed-day data yet") rather than sending a reversed range and letting it error, or worse, silently swapping the two dates and reporting the wrong period.

**⚠️ QTD/YTD/"since launch" frequently exceed the 90-day single-call cap — check the actual span before issuing one call.** A full quarter is 91-92 days on its own (already over the limit), so QTD past the first ~2 months of a quarter, any YTD beyond Q1, and "since launch" beyond ~3 months **all require the same split-then-merge procedure** as the aggregation-over-time example: split into contiguous ≤90-day windows first, then merge results by re-grouping on whatever time key you aggregated on and recomputing derived ratio metrics (`ACOS`/`CTR`/`CVR`/etc) from the summed base metrics — never average pre-computed ratios across windows. Don't assume "QTD" or "YTD" is safe to send as a single `dateStart`/`dateEnd` pair just because the phrasing sounds like one range — compute the actual day count first.
| Period-over-period / week-over-week (unspecified) | take an equal-length prior period, issue two `get_ads_perf` calls |
| Month-over-month (full calendar months, e.g. "July vs June") | compare the two full calendar months **as-is — do not force equal length** (months are 28-31 days); note the day-count difference to the user, and add each month's daily average if the user cares about rate rather than raw totals |
| Month-to-date-over-month-to-date (e.g. "this month so far vs last month so far") | align to the **same number of elapsed days** in each month (equal length is correct here, unlike full-calendar-month above) |

See the dedicated Period-over-Period and Top Movers example for the full procedure in all cases (join by ID, matching granularity/filters, full pagination before comparing, handling new/disappeared entities, zero-denominator deltas).

**`get_operation_log` has no documented processing delay** — operation logs record changes as they happen, not as a batch-aggregated report. Do **not** apply the same "default to yesterday" rule here.

| User says (get_operation_log) | Inferred range |
|---|---|
| Today's changes / what changed today | `dateStart` = `dateEnd` = **today** (not yesterday) |
| Last week | `dateStart` = last Monday, `dateEnd` = last Sunday |
| This month | `dateStart` = 1st of this month, `dateEnd` = **today** |
| Last 7 days | `dateStart` = 6 days ago, `dateEnd` = **today** |

## Common Pitfalls

- **`like` wildcard handling**: any `%` you include in a `like` pattern is stripped and replaced with an automatic leading+trailing `%` (case-insensitive substring match). Writing `{"like": "%test%"}` and `{"like": "test"}` behave the same — always effectively "contains", not prefix/suffix match.
- **Multi-profile amounts are USD** (except `asin.asinPrice_`/`asin` entity pricing, which stays local — see Currency Rules) — when you report numbers from a multi-profile `get_ads_perf`/`get_entity_metadata` call, say "in USD" explicitly so the user doesn't assume local currency.
- **Multi-profile rows need store attribution** — see Multi-Profile Row Attribution above; don't merge same-named campaigns across stores without labeling which store each row belongs to.
- **`get_operation_log` never paginates** — if `truncated=true`, split into non-overlapping date sub-windows and retry each until every sub-window returns `truncated=false`; if a single day alone is still truncated, say so rather than reporting a partial count as complete.
- **Query-param dates (`dateStart`/`dateEnd`) are `YYYY-MM-DD`; data-field dates (`date`, `campaignStartDate`, `campaignEndDate`) are `YYYYMMDD`** — see Date Format above. These are NOT interchangeable.
- **`get_ads_perf` max span is 90 days on the request itself** — week/month aggregation reduces row count within a call, it does NOT let one call's `dateStart`/`dateEnd` exceed 90 days. Split first, aggregate second.
- **ACOS/CTR/CVR/targetAcos/UnitSessionPercentage are confirmed pre-scaled ×100** (e.g. `17.61` = 17.61%) — don't re-scale, but **do append `%`** when presenting to the user (`"17.61%"`); filters stay on the raw ×100 scale (`{"ACOS": {"<": 20}}`). **TACOS/`*Rate` fields are unconfirmed** — relay the raw value as-is with no `%` and no assumed scale until backend confirms. See Ratio Metric Display Rule above.
- **Metric/field names are case-sensitive** — `Spend` ✓, `spend` ✗, `SPEND` ✗.
- **Field-naming convention differs by tool** — see the "Field Naming Rules" section in each tool's own SKILL.md before constructing `select`/`filters`. Do not assume the two tools share one convention.
