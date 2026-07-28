---
name: query-operation-log
version: 1.0.0
description: >-
  Query Xnurta platform operation logs: user and AI action history on ad entities.
  For tracking change history, auditing operations, troubleshooting issues.
  Keywords: operation log, change history, who changed what, bid adjustment records,
  budget adjustment, AI auto-adjustment, pause/enable records, operation audit,
  modification timeline, ad adjustment log
---

# Query Operation Log Skill

## MCP Tool

This skill maps to MCP tool: **`get_operation_log`**. Required scope: `amazon_sa:ads_logs:read`.

Profile, tenant, and user scope are resolved from the authenticated bearer token. `profileIds` is **required** — always call `get_user_authorized_context` first to obtain authorized profile IDs, then pass one or more into `profileIds`. Requested `profileIds` are intersected with the token's authorized set — an ID outside that set is silently dropped, not rejected (see Platform-Wide Rules below). If the user didn't specify a store, pass **all** authorized `profileIds`. Never pass `tenantId` or `userId`; the server derives them from the token.

**`userContext` (required)**: Must pass a non-empty string on every call. Preserve the user's original query as much as possible, plus the agent's reason for calling this tool. Summarize if too long, max 100 characters.

## Platform-Wide Rules

**Before using this tool, read [`references/platform-notes.md`](references/platform-notes.md)** — it covers auth flow, permission scopes, error handling, pagination/date-limit tables, currency rules, the tool-selection decision tree, and implicit inference rules shared across all 3 read tools. That file ships inside this same skill folder, so it travels with this skill regardless of how it's packaged/installed. This SKILL.md only covers what's specific to `get_operation_log`.

## ⚠️ This Tool Does NOT Support Pagination

This is the most important behavioral difference from the other two tools. `get_operation_log` is **limit-only**: it returns only the most recent N rows (time-descending), where N = `pageSize` (default 50, max 200). There is no `page` parameter at all. **Default to `pageSize: 200`** (the max) on every call, and **always check `truncated`** in the response. If `truncated=true`, your **first response should be to split the date range** into non-overlapping sub-windows and recurse — see "Getting a Complete Count" below for the exact procedure. Only narrow by `entities`/`resourceIds`/`changeBy`/`actionType`/`operationType` instead of splitting if you've already hit the single-day floor (a single day is still `truncated=true` on its own) or the user explicitly asked for one specific type of change — and if you do narrow by type as a substitute for splitting, say plainly that the result is a partial view, not complete history.

## When to Use

Use this tool when the user needs any of the following:
- Query ad operation history
- Find out who made what changes (manual vs AI)
- Track AI auto-adjustment records (bid, budget, etc)
- View bid/budget adjustment history
- Audit a specific campaign's change timeline
- Troubleshoot ad status changes (pause, enable, archive)
- Count how many changes of a given type happened, grouped by operator or entity (do this by pulling rows and aggregating client-side — the tool itself has no server-side `groupBy`)

See the Tool Selection Decision Tree above if the user's ask might belong to `get_ads_perf` or `get_entity_metadata` instead.

## DSL Parameter Format

```json
{
  "userContext": "User's original query + agent's reason for calling",
  "dateStart": "YYYY-MM-DD",
  "dateEnd": "YYYY-MM-DD",
  "profileIds": [1234567890123456],
  "entities": ["entity_type_list"],
  "resourceIds": [{"idEntity": "campaign", "ids": [123456]}],
  "changeBy": {"operator": "IN", "values": ["ai", "manual"]},
  "actionType": {"operator": "IN", "values": ["Bid Increased"]},
  "operationType": {"operator": "IN", "values": ["Campaign Paused"]},
  "campaignTypes": ["campaign_type_list"],
  "targetTypes": ["target_type_list"],
  "placementTypes": ["placement_type_list"],
  "pageSize": 200
}
```

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| profileIds | array[long] | **Yes** | — | Profile IDs from `get_user_authorized_context.profileIds`. Intersected with token's authorized set |
| dateStart | string | **Yes** | — | Start date `YYYY-MM-DD`. Max span vs `dateEnd` is 90 inclusive calendar days (see platform-notes.md for the precise off-by-one-safe definition); cannot be more than 15 months before today |
| dateEnd | string | **Yes** | — | End date `YYYY-MM-DD` |
| userContext | string | **Yes** | — | User's original query + reason, max 100 chars |
| entities | array[string] | No | all entities except `audience` | Entity type filter |
| resourceIds | array[object] | No | — | Resource ID filter (see below) |
| changeBy | object | No | — | Operator filter (see below) |
| actionType | object | No | — | Action type filter — coarse-grained (see below). ⚠️ **Only use values from the [actionType enum table](references/field-reference.md#actiontype)** — do not invent or guess values |
| operationType | object | No | — | Operation type filter — fine-grained (see below). ⚠️ **Only use values from the [complete operationType table](references/field-reference.md#complete-operationtype-values-by-entity)** — do not invent or guess values |

> **Recommended strategy**: when unsure which exact `operationType` values to use, first query with `actionType` (coarse-grained) to explore results and identify the specific `operationType` values appearing in the returned rows, then follow up with `operationType` (fine-grained) for precise filtering.
| campaignTypes | array[string] | No | — | Campaign type filter |
| targetTypes | array[string] | No | — | Targeting type filter (see below) |
| placementTypes | array[string] | No | — | Placement type filter (see below) |
| pageSize | int | No | 50 | Max rows returned, **max 200**. Results are time-descending; **there is no `page` parameter — this tool does not paginate** |

## Parameter Details, ChangeLogVO Field Reference

**See [`references/field-reference.md`](references/field-reference.md)** for the complete parameter enum reference (entities, resourceIds, campaignTypes, changeBy, actionType, operationType, targetTypes, placementTypes) and the ChangeLogVO response field table.

## Response Structure

```json
{
  "isError": false,
  "toolName": "get_operation_log",
  "rows": [
    {
      "entity": "campaign",
      "entityName": "Brand-SP-Auto-US",
      "amazonEntityId": 298539385213868,
      "entityId": 298539385213868,
      "operationType": "DailyBudget Increased",
      "profileId": 4404871489220462,
      "aiGroupName": "Growth-Group-1",
      "amazonCampaignId": 298539385213868,
      "campaignName": "Brand-SP-Auto-US",
      "campaignType": "sponsoredProducts",
      "amazonAdGroupId": null,
      "adGroupName": null,
      "changeField": "dailyBudget",
      "previousValue": "50.0",
      "newValue": "65.0",
      "countryCode": "US",
      "currencyCode": "USD",
      "createdDate": "2024-06-15 14:30:00",
      "changedBy": "ai"
    }
  ],
  "rowCount": 1,
  "limit": 50,
  "truncated": false,
  "effectiveProfileIds": [4404871489220462]
}
```

| Field | Type | Description |
|---|---|---|
| `isError` | boolean | Whether the call errored — check this before reading `rows` |
| `toolName` | string | Tool name |
| `rows` | array[ChangeLogVO] | Log entries, time-descending |
| `rowCount` | int | Number of rows returned |
| `limit` | int | The `pageSize` actually applied |
| `truncated` | boolean | **`true` means there were more matching records than `limit` could return.** Split the date range into non-overlapping sub-windows and continue querying each — do not try to page (there is no `page` param), and do not default to narrowing by type/entity filters, since that changes what you're searching for rather than just retrieving more of it. Only narrow by filter type as a last resort (a single day is still `truncated=true` on its own, or the user explicitly wants one type of change only) |
| `hint` | string | Guidance message, present when `truncated=true` |
| `effectiveProfileIds` | array[long] | Profile IDs actually applied after intersecting with the token's authorized set |

On error, the response instead follows the shared error envelope (`isError:true`, `errorType`, `message`, possibly `dimension`/`retryAfterSeconds`) described in Platform-Wide Rules above.

## Getting a Complete Count (handling `truncated=true` rigorously)

There is no server-side `groupBy` for this tool, and results are capped at `pageSize` (max 200) with no further pagination. To answer questions like "how many budget changes did each operator make" or "count of placement adjustments per campaign" **completely and correctly**, follow this procedure — a single narrowed retry is not sufficient to guarantee a correct count:

1. Issue the query with your filters (`entities`/`resourceIds`/`actionType`/`operationType`/`changeBy`) and check `truncated`.
2. **If `truncated=false`**: the returned rows are the complete result for that window — aggregate client-side (group by `changedBy`/`operationType`/etc and count) and you're done.
3. **If `truncated=true`**: split the date range into two **non-overlapping** sub-windows (bisect at the midpoint, with the second half starting the day after the first half ends — never let two sub-windows share a date), and recursively repeat this procedure on each half.
4. Only sum/merge counts across sub-windows that are non-overlapping — overlapping ranges will double-count entries in both.
5. **If a single day still returns `truncated=true`** on its own (more than 200 matching entries in one day for your filters), this tool **cannot guarantee a complete count** for that day. Don't report a partial number as if exact — tell the user the count is a lower bound (e.g. "at least 200 changes on 2024-06-15; can't return an exact count for a single day this active") and suggest narrowing by `resourceIds`/`operationType` if it matters.

## Examples

**Query last 7 days of AI auto-adjustment records:**
```json
{
  "profileIds": [4404871489220462],
  "dateStart": "2026-07-13",
  "dateEnd": "2026-07-19",
  "changeBy": {"operator": "IN", "values": ["ai"]},
  "pageSize": 200,
  "userContext": "Last 7 days of AI auto-adjustments"
}
```

**All operations on a specific campaign (resolve name to ID first via get_entity_metadata, then):**
```json
{
  "profileIds": [4404871489220462],
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "resourceIds": [{"idEntity": "campaign", "ids": [298539385213868]}],
  "pageSize": 200,
  "userContext": "Full change history for this campaign"
}
```

**Budget increases made manually by users (not AI/automation):**
```json
{
  "profileIds": [4404871489220462],
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "actionType": {"operator": "IN", "values": ["Budget Increased"]},
  "changeBy": {"operator": "IN", "values": ["manual"]},
  "pageSize": 200,
  "userContext": "Manual budget increases this month"
}
```

## Notes

- **`operationType` and `actionType` are closed enums** — you MUST only use values listed in [`references/field-reference.md`](references/field-reference.md). Never invent, guess, or interpolate values. If unsure whether a value exists, consult the reference table before sending the request. Values are English display strings (e.g. `"DailyBudget Increased"`, `"Campaign Paused"`) — pass them exactly as listed, case-sensitive
- `dateStart`/`dateEnd` request params use `YYYY-MM-DD`; `createdDate` in returned rows is a full UTC timestamp
- `dateEnd` must be equal to or later than `dateStart`
- Max date span is 90 days, max lookback is 15 months — these are hard limits, not auto-truncation. Split longer windows into multiple calls
- **This tool does NOT paginate.** There is no `page` parameter. `pageSize` (max 200) caps the total rows returned, always the most recent first. For a complete count over a large window, use the non-overlapping split-until-`truncated=false` procedure above — a single narrowed retry is not sufficient
- When `entities` is not specified, returns operations on all entity types **except `audience`**
- `profileIds` is **required**. Always call `get_user_authorized_context` first. If the user doesn't name a store, pass all authorized `profileIds`
- Requested `profileIds` are intersected with the authorized set, not rejected outright — check `effectiveProfileIds`
- `changeBy`, `actionType`, and `operationType` should be passed as objects with explicit `operator`/`values` (not bare arrays)
- To find a resource by name (campaign name, keyword text, ASIN) rather than ID, resolve it first via `get_entity_metadata`, then pass the ID into `resourceIds`
- No server-side aggregation (`groupBy`) — aggregate client-side after pulling rows, splitting non-overlapping date windows whenever `truncated=true`
- When `profileIds` spans multiple stores, map each row's `profileId` to a `profileName` before describing results
- Amounts in `previousValue`/`newValue` are always local currency — read the row's `currencyCode`, don't assume USD
- `placementTypes` has exactly 4 confirmed values — don't invent additional ones
- On error, check `errorType` and handle per the guidance above (e.g. `rate_limited` means wait `retryAfterSeconds` before retrying, not that the question is unanswerable)

## Reference Docs

- Shared cross-tool behavior (auth, errors, pagination, dates, currency, decision tree, inference rules): [`references/platform-notes.md`](references/platform-notes.md)
- Field dictionary (parameter enums, ChangeLogVO fields): [`references/field-reference.md`](references/field-reference.md)
- Enum i18n (ZH/EN/JA display labels for changeBy, actionType, operationType, entity — use this when presenting log entries to the user or translating between API values and localized display text): [`references/enum-i18n.md`](references/enum-i18n.md)
- Query examples:
  - [AI bid adjustment records (and how to broaden to all AI ops)](references/example-ai-bid-changes.md)
  - [All bid-change operations on a specific campaign](references/example-campaign-operations.md)
  - [SP budget changes (fine-grained filter)](references/example-user-budget-changes.md)
  - [Pause operations](references/example-pause-operations.md)
  - [Resolving a name to an ID before querying logs](references/example-resolve-name-to-id.md)
