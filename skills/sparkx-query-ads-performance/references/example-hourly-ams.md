# Hourly / AMS Queries (intraday and keyword × placement)

Hour-level data comes from Amazon Marketing Stream (AMS). Two entry points, both capped at a **7-day span** (not 90).

## 1. Campaign metrics by hour

```json
{
  "profileIds": [4404871489220462],
  "factEntity": "campaign",
  "timeGranularity": "hourly",
  "dateStart": "2026-08-17",
  "dateEnd": "2026-08-23",
  "select": ["campaign.campaignId_", "campaign.campaignName_", "date", "hour"],
  "metrics": ["Impressions", "Clicks", "Spend", "Sales", "Conversions"],
  "orderBy": [{"field": "Spend", "direction": "DESC"}],
  "pageSize": 500,
  "userContext": "Which hours of day drive spend and conversions last week"
}
```

Notes:
- `timeGranularity: "hourly"` is valid **only** with `factEntity: "campaign"`. On any other entity the call fails with `Parameter 'timeGranularity=hourly' is only supported for factEntity=campaign`.
- Span must be ≤ 7 days. `2026-08-17`→`2026-08-23` is exactly 7 inclusive days.
- `hour` must be present in `select`; `timeGranularity: "hourly"` does not add it automatically. With `date` and `hour` selected, build an intraday curve client-side by aggregating the same hour across days — and **divide by the number of days, not the number of rows**, when averaging.
- T+2 still applies: the newest hours are incomplete. Drop or label the trailing partial day rather than presenting it as a drop in performance.

## 2. Keyword × placement (SP only)

```json
{
  "profileIds": [4404871489220462],
  "factEntity": "keywordPlacement",
  "dateStart": "2026-08-17",
  "dateEnd": "2026-08-23",
  "select": ["keywordText", "matchType", "placement", "campaignId", "adGroupId"],
  "metrics": ["Impressions", "Clicks", "Spend", "Sales", "ACOS"],
  "orderBy": [{"field": "Spend", "direction": "DESC"}],
  "pageSize": 500,
  "userContext": "Top of search vs product page performance by keyword"
}
```

A returned row looks like this — note the keys are **not** the names you sent:

```json
{
  "keywordText_": "wireless earbuds",
  "matchType_": "EXACT",
  "placement": "Top of Search on-Amazon",
  "campaignId_": 12345,
  "adGroupId_": 67890,
  "hour": 20,
  "date": "20260823",
  "Spend": 18.4,
  "ACOS": 22.13
}
```

Notes:
- **Do not pass `timeGranularity`** here — this entity is hourly by nature, and it has no daily rollup either.
- **Naming is asymmetric**: send the bare names (`keywordId`, `keywordText`, `matchType`, `placement`, `campaignId`, `adGroupId`), read `_`-suffixed keys back (`keywordText_`, `matchType_`, …, plus `placement`, `hour`, `date`, `profileId_`). Don't feed a response key back into a request — `keywordPlacement.keywordText_` happens to resolve, but `keywordPlacement.placement_` fails because that column has no underscore.
- Enum values differ from the `placement` entity: `matchType_` is upper case (`EXACT`/`PHRASE`/`BROAD`) and `placement` is a display string (`Top of Search on-Amazon` / `Detail Page on-Amazon` / `Other on-Amazon`), not `topOfSearch`/`productPage`/`restOfSearch`.
- **SP only**, span ≤ 7 days. A `campaignType` filter is dropped here (SP-only source), so filtering for SB/SD returns an empty set that means "wrong entity", not "no data".
- **Metric set is restricted**: base metrics, Same/Other-SKU variants, and `CTR`/`CVR`/`CPC`/`CPA`/`ACOS`/`ROAS`. Requesting an NTB / DPV / video / viewability / impression-share / ASIN-business metric **fails the call** — that column doesn't exist here.
- **AI metrics are placeholders on this entity**: `AISpend`/`AISales` return `0`, `AIACOS`/`AIROAS` return `null` (accepted, but meaningless). Never report them as real values and never use them as a denominator — switch to `factEntity: campaign` for AI attribution.

## When the user wants a longer hourly window

"Show me the hourly pattern for last month" cannot be one call. Options, in order of preference:

1. Aggregate hour-of-day across several ≤7-day calls and **tell the user you sampled** (e.g. "based on the last 4 weeks, one week per call").
2. Fall back to daily granularity and state that hourly data is limited to a 7-day window.

Don't silently truncate the range to 7 days and present it as the month.
