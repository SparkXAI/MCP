# Action-space coupling rules (SP/SB `aiActionSettings`)

`aiActionSettings` is not just on/off switches - several switches are meaningless (or
fail / fall back to unintended defaults) unless you also send their companion fields.
When enabling one, include the whole set. These are front-end-enforced linkages; MCP
won't enforce them, so you must. (Exact field names: `field-reference.md`.)

- **Dynamic budget** (`budgetDynamicActionStatus=1`) -> also `budgetNumType`
  (`1`=percentage / `2`=fixed) and `budgetNum` (the value). Value bounds (the batch path
  uses `budgetDynamicStatus`/`numType`/`num`): `num > 0`; `numType=1` (percentage)
  `num <= 1000`; `numType=2` (fixed) `num <= 100000`.
- **Bid range** (`bidRangeStatus=1`) -> also `bidRangeType` and `bidRange` `[min,max]`
  (array length exactly 2, **min <= max**). `bidRangeType=1` (percentage): `0.01 <= min`
  and `max <= 100`. `bidRangeType=2` (fixed): `min >=` the site's minimum bid. Note:
  **BidDaypart and BidAdjustment share this one `bidRange`** - if both are on it applies
  to both.
- **B2B range** (`btbRangeStatus=1`) -> `btbMin`/`btbMax`: `btbMin >= 0`, `btbMax <= 900`,
  min <= max, at least one end non-null.
- **Placement ranges** (`bidAdPlaceRangeStatus=1`) -> **at least one** of the `tos*`/
  `pdp*`/`ros*` min/max pairs non-null (don't enable the switch with all null); each pair
  `min >= 0`, `max <= 900`, min <= max.
## Word-list fields are not supported

All word-list settings are currently unsupported, including branded, non-branded,
competitor, negative-target blacklist, and target-harvest blacklist settings. **Do not
send their status, list ID, match-type, or list-type fields for SP, SB, or SD**, even if
the routed schema exposes them. Tell the user to configure word lists in the platform.

## Disabling an action space doesn't clear its follow-up fields

Turning an action space off (its `*Status=0`) only updates that status - the backend does
**not** zero out the range/companion values you set before (confirmed 2026-08-14). To
change a follow-up value you must re-send it.
