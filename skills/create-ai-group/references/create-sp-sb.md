# Create SP/SB managed group - `save_sp_sb_ai_managed_group` (create mode)

Creates a **Sponsored Products** or **Sponsored Brands** AI managed group. All
arguments go inside a single `request` object. **Create mode is triggered by leaving
`aiGroupId` empty (omit it or set `0`)** - a positive `aiGroupId` means edit, which
belongs to the `edit-ai-group` skill.

Create defaults to **AI mode**: `aiActionSettings.xxxStatus=1` (switch on) +
the corresponding `aiAutomation` mode field = `0` (AI auto-decision). You can set
Rule mode (mode field = `1`) at create time, but Rule condition/action configs
(the 7x24 matrix, etc.) are not editable through this tool — only through the
platform UI. So Rule mode at create uses platform-default templates. The exact
mode field names per action space are in
[`field-reference.md`](field-reference.md) under `aiAutomation`.

## Required for create

| Field | Type | Notes |
|---|---|---|
| `profileId` | long | Shop ID |
| `aiGroupName` | string | Unique per profile |
| `acos` | number | Target ACOS on the **x100 scale** - the user's `25%` is sent as **`25`** (not `0.25`). Must be `> 0` |
| `targetType` | int | `1`=drive growth, `2`=maintain stability, `3`=volume, `4`=legacy growth |
| `aiStatus` | int | `0`=off, `1`=on |
| `campaignType` | string | `"sponsoredProducts"` or `"sponsoredBrands"` |

## Common optional fields

| Field | Type | Notes |
|---|---|---|
| `campaignIds` | int[] | Campaigns to include (auto-increment IDs). Omit to create an empty group |
| `campaignNameSign` | int | Campaign-name label: `0`=off, `1`=on |
| `aiPersonality` | int | `1`-`5`; **must be >=3 when `targetType=3` (volume/冲量)** (front-end rule - MCP won't enforce it) |
| `preAddCampaignNums` | int | Pre-add campaign count |
| `aiAutomation` | object | Mode fields: `0`=AI mode, `1`=Rule mode. Exact field names in field-reference |
| `aiActionSettings` | object | Action-space config (bid / struct / budget / target / brand optimization). See field-reference |

Only send `aiAutomation` / `aiActionSettings` fields the user wants to change from
platform defaults - both are large flattened objects and you rarely need most of it
for a basic create. Exact field names are in `field-reference.md`.

## Action-space coupling rules

Enabling an action-space switch usually requires sending its companion fields (dynamic
budget -> numType+num; ranges -> min/max with min <= max; branded/competitor -> list +
match type; blacklists -> list + list-type + match-type). The full set is in the shared
[`coupling-rules.md`](coupling-rules.md) - read it before enabling any `aiActionSettings`
switch.

## SP vs SB differences (SB is not SP)

Some capabilities exist only for SP. If the group is SB, do **not** send these -
per the 2026-08-14 backend spec they should be rejected, but today they may instead be
silently ignored (looks successful, does nothing):

- `aiActionSettings.structPauseProductStatus` / `structPauseCampaignStatus`
  (struct optimization) - **SP only**
- `aiActionSettings.targetPausedAddStatus` - **SP only**

When the user is on SB and asks for one of these, tell them it isn't available for
Sponsored Brands rather than sending it quietly. The full "which capability is
supported (AI / Rule / none) per SP / SB / SD" matrix is in
[`action-space-matrix.md`](action-space-matrix.md) - check it before enabling any
action-space switch (notably: **SB's BidDaypart has no AI mode**, and
`budgetRedistribute` / `bidAmazonBusiness` are `noRule` - never attach Rule configs).

## Example - create a basic SP group, AI off

```json
{
  "request": {
    "profileId": 3721212165742,
    "aiGroupName": "SP-Core-Keywords-US",
    "campaignType": "sponsoredProducts",
    "acos": 25,
    "targetType": 1,
    "aiStatus": 0,
    "campaignIds": [45444534]
  }
}
```

`campaignIds` uses each campaign's internal `campaignId` (int) - **not**
`amazonCampaignId` (the long Amazon string).

Success returns `data.result.aiGroupId`. Then verify with the full-signature read
`get_entity_metadata(profileIds=[3721212165742], entity='aiGroup',
filters={"aiGroupName": {"like": "%SP-Core-Keywords-US%"}}, userContext='verify created group')`
and exact-match the name in the results (the filter is a substring `like`).

> Reminder: a group created with `aiStatus=0` reads back as `aiStatus=2`
> ("AI Turned Off") - that's expected, it is off.
