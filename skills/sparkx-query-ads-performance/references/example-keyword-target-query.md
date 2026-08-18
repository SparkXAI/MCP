# Keyword Target Query

Find enabled keyword targets with the lowest ACOS (spend above a floor, to avoid noisy near-zero-spend keywords):

```json
{
  "profileIds": [4404871489220462],
  "factEntity": "target",
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "metrics": ["Spend", "Sales", "ACOS", "Clicks"],
  "select": ["target.targetText_", "target.targetMatchType_"],
  "queryType": "keyword",
  "filters": {"target.targetState_": "enabled", "Spend": {">": 5}},
  "orderBy": [{"field": "ACOS", "direction": "ASC"}],
  "userContext": "Keywords with the lowest ACOS"
}
```

Notes:
- `queryType: "keyword"` is required whenever `factEntity` is `target` (or `searchTerm`). On `target`, `queryType` can be `keyword`/`product`/`auto`; on `searchTerm` only `keyword`/`product` are valid (`auto` is not).
- `target.targetMatchType_` for a keyword-type query returns `broad`/`phrase`/`exact`.
- `Spend": {">": 5}` is a metric-field filter (evaluated in HAVING), separate from the dimension-field filter `target.targetState_` (evaluated in WHERE) — both can appear in the same flat `filters` object.
