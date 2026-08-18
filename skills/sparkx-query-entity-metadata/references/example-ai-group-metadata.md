# AI Group Metadata Query

List currently-running AI managed groups.

```json
{
  "profileIds": [4404871489220462],
  "entity": "aiGroup",
  "filters": {"aiStatus": 1},
  "userContext": "List currently-enabled AI managed groups"
}
```

**With a name filter and sort**:

```json
{
  "profileIds": [4404871489220462],
  "entity": "aiGroup",
  "filters": {
    "AND": [
      {"aiGroupName": {"like": "%growth%"}},
      {"aiStatus": 1}
    ]
  },
  "orderBy": [{"field": "aiGroupName", "direction": "ASC"}],
  "userContext": "AI groups matching 'growth'"
}
```

Notes:
- Filters (`aiStatus`, `aiGroupName`) use `get_entity_metadata`'s camelCase convention (no `aiGroup.` prefix, no `_` suffix) — different from `get_ads_perf`.
- `targetAcos` (if returned — this tool has no `select` param, so which fields appear depends on the entity type, not on a field list you choose) is Tier 1 confirmed ×100/percentage — append `%` when presenting it.
- Unlike `get_ads_perf`, this tool's spec does not document support for custom SQL aggregate expressions (e.g. `count(distinct ...)`) in `select` — stick to the plain fields listed in the `aiGroup` field table.
