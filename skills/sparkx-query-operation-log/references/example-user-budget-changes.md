# SP Budget Changes (Fine-Grained operationType Filter)

```json
{
  "profileIds": [4404871489220462],
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "campaignTypes": ["sponsoredProducts"],
  "operationType": {"operator": "IN", "values": ["DailyBudget Increased", "DailyBudget Decreased"]},
  "pageSize": 200,
  "userContext": "SP budget adjustment records"
}
```

**Same intent, but restricted to manual (human) changes only** — combine `operationType` with `changeBy`:

```json
{
  "profileIds": [4404871489220462],
  "dateStart": "2026-06-01",
  "dateEnd": "2026-06-30",
  "campaignTypes": ["sponsoredProducts"],
  "operationType": {"operator": "IN", "values": ["DailyBudget Increased", "DailyBudget Decreased"]},
  "changeBy": {"operator": "IN", "values": ["manual"]},
  "pageSize": 200,
  "userContext": "Manual SP budget adjustments only, excluding AI/automation"
}
```

Note: `operationType` (fine-grained) is more precise than `actionType` (coarse categories like `"Budget Increased"`) — use `operationType` when you need to isolate a specific sub-case.

**Check `truncated` before presenting either result as complete** — if `true`, split the date range into non-overlapping sub-windows and repeat, rather than presenting a partial result as the full history.
