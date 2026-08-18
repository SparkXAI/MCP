# Action-space coupling rules (SP/SB `aiActionSettings`)

`aiActionSettings` switches (`xxxStatus`) are **action-space on/off**: `0`=off
(disabled), `1`=on (active). When on, the mode (AI vs Rule) is set by the
corresponding `aiAutomation` mode field: `0`=AI, `1`=Rule. The exact mode field
name per action space is in [`field-reference.md`](field-reference.md). Several
switches also
require companion fields (ranges, lists, num) to be meaningful — these coupling rules
apply regardless of AI or Rule mode. When enabling a switch, include the whole set.
These are front-end-enforced linkages; MCP won't enforce them, so you must. (Exact
field names: `field-reference.md`.)

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
- **Branded mode** (`brandedStatus=1`) -> `brandedList` (word-list IDs) and
  `brandedMatchType` (`1`=exact / `2`=phrase). **Competitor** (`competitorStatus=1`) ->
  `competitorList` + `competitorMatchType`.

Branded/competitor list IDs (`brandedList` / `competitorList`) can't be invented - the
user must supply them (or pick from the UI). If a required list isn't available, tell the
user rather than sending an empty/guessed one.

## Word-list fields: branded/competitor OK, blacklists NOT supported

Keep these two apart:
- **Branded / competitor** (`brandedStatus` / `competitorStatus` + lists) are **not
  version-gated** - the backend doesn't filter by group version (v1/v2 is UI-only), and
  prod testing (2026-08-13) confirmed they're accepted and take effect on v1. Send them
  normally; list IDs must be user-supplied.
- **Blacklist word-lists** (`negativeTargetBlackListStatus` / `targetHarvestBlackListStatus`,
  the "don't-negate" / "don't-harvest" keyword lists) are **currently NOT supported for
  any ad type** (2026-08-14 backend spec: should be rejected). **Do not send
  `*BlackListStatus=1` or the `*BlackList` IDs for SP, SB, or SD** - the backend may not
  block it yet (a bypassable gap) but it has no effect. Tell the user it isn't available
  through MCP.

## Disabling an action space doesn't clear its follow-up fields

Turning an action space off (`aiActionSettings.xxxStatus=0`) disables it entirely
(neither AI nor Rule runs). It does **not** zero out the range/companion values you
set before (confirmed 2026-08-14), and it does **not** clear the corresponding
`aiAutomation` mode field or its Rule template (the Rule template remains). To
change a follow-up value you must re-send it.
