# ASIN Business Metrics Query

Use `factEntity: "asin"` to get ASIN-level business metrics (TotalSalesAmount, TACOS, Sessions, etc.) alongside standard ad metrics.

```json
{
  "profileIds": [4404871489220462],
  "factEntity": "asin",
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "metrics": ["Spend", "Sales", "ACOS", "TotalSalesAmount", "OrderCount", "TACOS"],
  "select": ["asin.asin_", "asin.parent_asin_", "asin.asinTitle_"],
  "orderBy": [{"field": "TotalSalesAmount", "direction": "DESC"}],
  "pageSize": 20,
  "userContext": "Top 20 ASINs by total sales, with ad spend/ACOS and TACOS"
}
```

Notes:
- `TotalSalesAmount`, `OrderCount`, `TACOS`, and the rest of the ASIN business-metrics block (`Sessions`, `BuyBoxPercentage`, `OrganicSales`, etc.) are **only valid on the `asin` fact entity** — requesting them on `campaign`/`adGroup`/etc returns an error.
- `TACOS` is a Tier 2 (unconfirmed scale) metric — relay its raw value as-is, do not assume it's ×100 and do not append `%`, unlike `ACOS`.
- If querying multiple profiles, `asin.asinPrice_` (if selected) stays local currency even though other monetary metrics convert to USD.
