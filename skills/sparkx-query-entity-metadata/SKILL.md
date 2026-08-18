---
name: sparkx-query-entity-metadata
version: 1.1.0
description: >-
  Query advertising entity metadata: name, status, configuration, settings (no time range or metrics).
  For finding entity lists, getting config details, discovering ad structure relationships.
  Keywords: campaign list, campaign name, status query, AI group settings, ASIN info,
  ad group config, portfolio list, keyword list, product line, entity search,
  which ads, enabled/paused status, budget settings, bidding strategy, automation rules
---

# Query Entity Metadata Skill

## MCP Tool

This skill maps to MCP tool: **`get_entity_metadata`**. Required scope: `amazon_sa:ads_configuration:read`.

Profile scope is resolved from the authenticated bearer token. `profileIds` is **required** — always call `get_user_authorized_context` first to obtain authorized profile IDs, then pass one or more into `profileIds`. Requested `profileIds` are intersected with the token's authorized set — an ID outside that set is silently dropped, not rejected (see Platform-Wide Rules below). If the user didn't specify a store, pass **all** authorized `profileIds`. Never pass `tenantId` or `userId`; the server derives them from the token.

**`userContext` (required)**: Each call must include a non-empty string. Preserve the user's original query as much as possible, plus the agent's reason for calling this tool. Summarize if too long, max 100 characters.

## Platform-Wide Rules

**Before using this tool, read [`references/platform-notes.md`](references/platform-notes.md)** — it covers auth flow, permission scopes, error handling, pagination tables, the Ratio Metric Display Rule (including `targetAcos`), currency rules, the tool-selection decision tree, and implicit inference rules shared across all 3 read tools. That file ships inside this same skill folder, so it travels with this skill regardless of how it's packaged/installed. This SKILL.md only covers what's specific to `get_entity_metadata`.

## When to Use

Use this tool when the user needs any of the following:
- Query entity metadata (name, status, config, etc)
- Find entity lists (e.g. all enabled campaigns, AI group list, portfolios with a budget set)
- Get entity configuration info (budget settings, bidding strategy, AI group targets, current keyword bid)
- Discover ad structure relationships (which AdGroups under a Campaign, ASIN ownership, which automation rules are enabled on a campaign)
- Search entities by condition (name fuzzy match, status filter, bid/budget threshold filter)
- Count entities (e.g. how many campaigns, how many portfolios have a budget)

**⚠️ Counting entities requires paging through all results, not just reading page 1's `rowCount`.** `rowCount` is the number of rows on the *current page only* — it is not a total count. To answer "how many X are there", loop `page` (incrementing each call) while `hasNextPage` is `true`, summing each page's `rowCount` as you go; the running total once `hasNextPage` is `false` is the real count. Consider raising `pageSize` to the max (500) first to reduce the number of round trips. Do not report a page-1 `rowCount` as if it were the full answer.

**Note**: This tool does NOT involve time range or performance metrics. For spend, clicks, ACOS etc, use `get_ads_perf`.

**Note**: This tool does NOT return historical point-in-time snapshots — it only returns the entity's *current* configuration. There is no `asOfDate`-style parameter, and this tool has no date parameters at all. If the user asks "what was the daily budget on 2026-03-26", that specific historical value cannot be retrieved with this tool; only the current value is available. Say so explicitly rather than guessing.

See the Tool Selection Decision Tree above if the user's ask might belong to `get_ads_perf` or `get_operation_log` instead.

## Required Parameter: `entity`

Unlike `get_ads_perf` (which infers tables from `select`), `get_entity_metadata` requires you to **explicitly declare which entity you're querying** via the `entity` parameter. Omitting it is the most common cause of a failed call (`errorType: invalid_params`).

## DSL Parameter Format

```json
{
  "userContext": "User's original query + agent's calling reason",
  "entity": "campaign",
  "profileIds": [1234567890123456],
  "filters": {},
  "orderBy": [{"field": "fieldName", "direction": "ASC"}],
  "select": ["campaignId", "campaignName", "campaignState"],
  "page": 1,
  "pageSize": 100
}
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| profileIds | array[long] | **Yes** | — | Profile IDs from `get_user_authorized_context.profileIds`. Intersected with token's authorized set |
| entity | string | **Yes** | — | Entity type to query. See enum table below. Note: productLine info is nested inside `asin` entity results, not a separate `entity` value |
| userContext | string | **Yes** | — | User's original query + reason, max 100 chars |
| filters | object | No | {} | Filter conditions. Field names are **camelCase**, no `entity.` prefix, no `_` suffix (e.g. `campaignState`, not `campaign.campaignState_`) |
| orderBy | array[object] | No | [] | `[{"field": "fieldName", "direction": "DESC"}]` |
| select | array[string] | No | all fields | Return **only** these top-level fields (no nested paths), in the given order. Unknown fields are ignored (reported via `meta.hint`). Does not affect pagination. See "Trimming fields with `select`" below |
| page | int | No | 1 | Page number (1-based) |
| pageSize | int | No | 100 | Rows per page, max 500 |

## Trimming fields with `select`

When you only need a few fields, pass `select` to return just those — it noticeably shrinks the response and saves tokens. Rules:

- Field names use this tool's **plain camelCase** (e.g. `campaignState`) — **not** `get_ads_perf`'s `entity.field_` format.
- **Top-level fields only** — nested paths are not supported.
- Output order follows `select`; fields absent from a row are simply omitted; unknown field names are ignored and surfaced in `meta.hint` (with the first row's available fields).
- `select` **does not affect pagination** (`page`/`pageSize`/`hasNextPage` behave the same).
- ⚠️ **The `{field}Text` companion fields are NOT auto-included under `select`.** `select` is a strict projection: only the fields you list come back. If you still need the human-readable enum label (or will translate enums), you **must list both** the field and its `Text` companion, e.g. `["campaignState", "campaignStateText"]`. Selecting only `campaignState` returns the raw value `"paused"` — you won't get `"Paused"`.

## ⚠️ Field Naming Differs From `get_ads_perf`

This is the single most important thing to get right. `get_ads_perf` dimension fields use `entity.field_` (prefix + underscore suffix). **`get_entity_metadata` fields do NOT** — they are plain camelCase field names with no prefix and no suffix:

| Tool | Example |
|---|---|
| `get_ads_perf` (select/filters) | `campaign.campaignState_` |
| `get_entity_metadata` (filters/orderBy) | `campaignState` |

Do not copy the `get_ads_perf` field format into this tool — it will fail.

## `entity` Enum Values, Fields by Entity

**See [`references/field-reference.md`](references/field-reference.md)** for the complete field dictionary: entity enum values and backing providers, and all per-entity field tables (profile, campaign, adGroup, target, productAd, portfolio, placement, aiGroup, asin, automationRule) with filterable fields, enums, and usage examples.

## Filter Syntax

Field names are **camelCase, no prefix/suffix** (different from `get_ads_perf`!).

```json
{
  "campaignState": "enabled",
  "campaignId": [123, 456],
  "campaignName": {"like": "%test%"},
  "AND": [
    {"campaignState": "enabled"},
    {"campaignName": {"like": "%brand%"}}
  ],
  "OR": [
    {"campaignState": "enabled"},
    {"campaignState": "paused"}
  ]
}
```

**Supported operators**:

| Operator | Meaning | Backend op | Usage |
|---|---|---|---|
| `=` (bare value) | Equals | eq | `{"campaignState": "enabled"}` |
| `!=` | Not equals | ne | `{"campaignState": {"!=": "archived"}}` |
| `like` | Fuzzy match | like (case-insensitive) | `{"campaignName": {"like": "%test%"}}` |
| `in` | In list | in | `{"campaignState": {"in": ["enabled", "paused"]}}` |
| `notin` | Not in list | notin | `{"campaignState": {"notin": ["archived"]}}` |
| `>=`, `<=` | Range | between | `{"dailyBudget": {">=": 10, "<=": 100}}` |
| `>`, `<` | Range | between | `{"defaultBid": {">": 0.5, "<": 2}}` |

**Note on `like`**: any `%` you include is stripped and replaced with an automatic leading+trailing `%` — the match is always "contains" regardless of where you place `%`.

## Enum "Text" Companion Fields

For enum-valued fields, the response **automatically appends a human-readable `{field}Text` companion field** — but **only when you do NOT use `select`**. If you pass `select`, this auto-append does not happen; you must list each `xxxText` field explicitly (see "Trimming fields with `select`" above). Example (no `select`):
```json
{
  "campaignState": "enabled",
  "campaignStateText": "Enabled",
  "campaignType": "sponsoredProducts",
  "campaignTypeText": "Sponsored Products",
  "biddingStrategy": "autoForSales",
  "biddingStrategyText": "SP Dynamic bids - up and down"
}
```

Fields that get this treatment: all `*State` fields (campaignState/adGroupState/targetState/portfolioState/productAdState), all `*ServingStatus` fields, `campaignType`, `biddingStrategy`, `targetingType`/`targetMatchType`/`matchType`, `placement`, `costType`/`budgetType`/`portfolioBudgetType`, `aiStatus`/`aiTargetType`/`aiPersonality`, `asinInventoryStatus`/`asinSpEligibilityStatus`/`asinIsDelete`, generic `xxxStatus` (0/1) flags, `countryCode`, `isAiCreate`/`sdBidOptimization`/`profileUseBudgetCap`.

**Exception**: `automationRule.enabledRuleNames` is already a human-readable string array — there is no separate `Text` field for it.

## Response Structure

```json
{
  "isError": false,
  "toolName": "get_entity_metadata",
  "rows": [
    {
      "campaignId": 826117,
      "amazonCampaignId": "298539385213868",
      "campaignName": "Brand-SP-Auto-US",
      "campaignType": "sponsoredProducts",
      "campaignState": "enabled",
      "biddingStrategy": "autoForSales",
      "targetingType": "auto",
      "dailyBudget": 50.0,
      "currentBudget": 50.0,
      "portfolioId": "12345",
      "aiGroupId": "501"
    }
  ],
  "rowCount": 1,
  "page": 1,
  "pageSize": 100,
  "hasNextPage": false,
  "effectiveProfileIds": [4404871489220462]
}
```

| Field | Type | Description |
|---|---|---|
| `isError` | boolean | Whether the call errored — check this before reading `rows` |
| `toolName` | string | Tool name |
| `rows` | array[object] | Result rows — fields depend on `entity` |
| `rowCount` | int | Row count on current page |
| `page` / `pageSize` | int | Pagination state |
| `hasNextPage` | boolean | Whether more pages exist |
| `effectiveProfileIds` | array[long] | Profile IDs actually applied after intersecting with the token's authorized set |

On error, the response instead follows the shared error envelope (`isError:true`, `errorType`, `message`) described in Platform-Wide Rules above. A missing `entity` param surfaces as `errorType: invalid_params`.

## Notes

- `entity` is **required** — this is the #1 cause of failed calls. Do not call this tool without it.
- Each page returns up to `pageSize` rows (max 500); use `page` for pagination and check `hasNextPage`. Exception: `automationRule` ignores pagination entirely.
- `profileIds` is **required**. Always call `get_user_authorized_context` first. If the user doesn't name a store, pass all authorized `profileIds`
- Requested `profileIds` are intersected with the authorized set, not rejected outright — check `effectiveProfileIds`
- No `dateStart`/`dateEnd` or `metrics` needed — and no historical/point-in-time snapshot capability either (no date params of any kind on this tool)
- `groupBy` is not applicable here — this tool returns entity rows, not aggregates
- User says "ASIN" -> default to child ASIN, use entity `asin`, field `asin`
- `profile` **can** be queried standalone as its own entity
- `automationRule` **requires** `amazonCampaignId` in filters, and does not support sort/pagination
- `campaignStartDate`/`campaignEndDate` use `YYYYMMDD` (Ymd) — different from the `YYYY-MM-DD` used by `dateStart`/`dateEnd` on the other two tools
- For campaign rows, `campaignId` is the internal integer ID used by managed-group write tools; `amazonCampaignId` is the Amazon ID used to link performance/log data. Never substitute one for the other
- `asin` entity's currency handling is the one exception to "multi-profile = USD" — check each row's `currency` field
- When querying across multiple `profileIds`, verify whether the entity's rows carry a `profileId` field (e.g. `asin` does); if not, query per-profile or cross-reference before merging
- `targetAcos` (aiGroup entity) is confirmed ×100/percentage, same as performance `ACOS` — don't re-scale, but append `%` when presenting it
- Only use field names listed in this doc's per-entity tables; never invent field names
- Field naming here (camelCase, no prefix/suffix) is **different** from `get_ads_perf` (`entity.field_`) — do not mix the two conventions
- On error, check `errorType` and handle per the guidance above

## Reference Docs

- Shared cross-tool behavior (auth, errors, pagination, currency, ratio-metric display rule, decision tree, inference rules): [`references/platform-notes.md`](references/platform-notes.md)
- Field dictionary (entity enum values, per-entity field tables, filterable fields, enums): [`references/field-reference.md`](references/field-reference.md)
- Enum i18n (ZH/EN/JA display labels for all enum values — use this when presenting enum fields to the user or translating between API values and localized display text): [`references/enum-i18n.md`](references/enum-i18n.md)
- Query examples:
  - [Basic entity list query](references/example-meta-only.md)
  - [Filtered campaign list](references/example-campaign-filter.md)
  - [AI group metadata query](references/example-ai-group-metadata.md)
  - [ASIN metadata (with parent ASIN + product line)](references/example-asin-metadata.md)
  - [Automation rule lookup](references/example-automation-rule.md)
