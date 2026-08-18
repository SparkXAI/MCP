# Automation Rule Lookup

Which automation rules are enabled on a set of campaigns:

```json
{
  "profileIds": [4404871489220462],
  "entity": "automationRule",
  "filters": {"amazonCampaignId": {"in": [123456789, 987654321, 555555555]}},
  "userContext": "Which automation rules are enabled on these campaigns"
}
```

Response:
```json
{
  "isError": false,
  "toolName": "get_entity_metadata",
  "rows": [
    {"amazonCampaignId": 123456789, "enabledRuleTypes": [2, 4], "enabledRuleNames": ["Dayparting", "Harvest Keywords"]},
    {"amazonCampaignId": 987654321, "enabledRuleTypes": [], "enabledRuleNames": []},
    {"amazonCampaignId": 555555555, "enabledRuleTypes": [17, 19], "enabledRuleNames": ["Budget Performance", "Placement Rule"]}
  ],
  "rowCount": 3,
  "page": 1,
  "pageSize": 3,
  "hasNextPage": false,
  "effectiveProfileIds": [4404871489220462]
}
```

Notes:
- `amazonCampaignId` in `filters` is **required** for this entity — there is no way to list all campaigns' rules without specifying which campaigns.
- No `orderBy`/`page`/`pageSize` effect — results always come back in the order the campaign IDs were requested.
- An empty campaign in the response (`enabledRuleTypes: []`) means that campaign simply has no automation rules enabled, not an error.
