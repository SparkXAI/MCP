# Search Term Query

`searchTerm` is the one entity whose dimension fields drop the `entity.` prefix but keep the `_` suffix — a common source of mistakes.

```json
{
  "profileIds": [4404871489220462],
  "factEntity": "searchTerm",
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "metrics": ["Spend", "Sales", "Conversions", "CTR"],
  "select": ["query_", "matchType_", "campaign.campaignName_"],
  "queryType": "keyword",
  "filters": {"campaign.campaignId_": 298539385213868},
  "orderBy": [{"field": "Spend", "direction": "DESC"}],
  "userContext": "Search term report for a specific campaign, sorted by spend"
}
```

Notes:
- `query_`/`matchType_` (searchTerm's own fields) have **no entity prefix** — not `searchTerm.query_`.
- `campaign.campaignName_`/`campaign.campaignId_` (joined dimension) **do** keep the normal `entity.field_` form, since they belong to the `campaign` entity, not `searchTerm`.
- `queryType` is required here too, same as for `target` — but on `searchTerm` only `keyword`/`product` are valid (`auto` is not, unlike `target`).
- `CTR` in the response is Tier 1 confirmed pre-scaled ×100 — append `%` when presenting it.
