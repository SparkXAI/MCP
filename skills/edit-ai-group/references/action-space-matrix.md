# Action-space support matrix (SP / SB / SD)

Not every action-space capability is available for every ad type, and some support a
different **mode** (AI auto-decision vs Rule-based). The platform UI hides the
unsupported ones; **MCP does not** - so before enabling any action-space switch, check
it's actually supported for this group's `campaignType`. If it isn't, tell the user
it's not supported for that ad type and don't send it (a silently-ignored switch looks
like success but changes nothing).

> Create defaults to **AI mode** (`aiActionSettings.xxxStatus=1` +
> the corresponding `aiAutomation` mode field = `0`). Rule mode (mode field = `1`)
> can be set at create time, but Rule condition/action configs (the 7x24 matrix,
> etc.) are not editable through the write tool — only through the platform UI.
> The exact mode field name per action space is in
> [`field-reference.md`](field-reference.md).

## Platform support != what the write tool exposes

The table below is **platform-level** capability support. It does **not** mean the MCP
write tool for that ad type actually has a field for it. Two layers must both hold to
make a valid call: **platform supports it** AND **the tool exposes a field for it**.

The gap that matters most: **the SD tools (`create_sd_ai_managed_group` /
`edit_sd_ai_managed_group`) expose only a small set** - `status`, `optimizeType`, `acos`
(edit adds `acosType`/`roasType`/`budgetType` + values), `budgetDynamicStatus`/
`numType`/`num`, `budgetRedistributeStatus`, `targetHarvestStatus`, `aiPersonality`,
`campaignNameSign` (edit: `campaignNameRecoveryType`, `remark`). They do **not** expose
the full `aiActionSettings`/`aiAutomation` action space (bid daypart / placement /
negative target / struct pause / branded-competitor / etc.). Those fields exist **only
on the SP/SB tool** (`save_sp_sb_ai_managed_group`).

**SD supports only two of these action spaces** - `预算重新分配` (budgetRedistribute)
and `定向收割` (targetHarvest). Everything else in the table below is SD N. And even for
those two, confirm the SD write tool actually exposes a field before using them.

Source of truth: the per-ad-type **"AI 行动空间设置" UI** - what each ad type actually
renders. If an action space isn't in that ad type's UI, it's unsupported for that type;
the UI hides it and **MCP silently ignores the field** if you send it.

**Definitive per-ad-type support:**
- **SP - all 12 (AI):** every row below.
- **SB - 7:** 分时调价 *(Rule only)*, 按表现调价, 分时预算, 按表现调预算, 预算重新分配,
  添加否定定向, 定向收割.
- **SD - 2:** 预算重新分配, 定向收割.

| 行动空间 (UI) | field family (rule#) | SP | SB | SD |
|---|---|---|---|---|
| 广告位调价 (placement bids) | `bidAdPlace` / placementAdjustment (19) | Y | N | N |
| 分时调价 (bid dayparting) | `bidDaypart` (2) | Y | Y *(SB: Rule only, no AI)* | N |
| 按表现调竞价 (bid by performance) | `bidPerformance` (181) | Y | Y | N |
| B2B调价 | `bidAmazonBusiness` (noRule) | Y | N | N |
| 分时预算 (budget dayparting) | `budgetDaypart` (13) | Y | Y | N |
| 按表现调预算 (budget by performance) | `budgetPerformance` / `budgetDynamicAction` (17) | Y | Y | N |
| 预算重新分配 (budget reallocation) | `budgetRedistribute` (noRule) | Y | Y | Y |
| 添加否定定向 (add negative target) | `negativeTarget` (5) | Y | Y | N |
| 定向收割 (target harvest) | `targetHarvest` (4) | Y | Y | Y |
| 定向暂停 (target pause) | `targetPausedAdd` / targetPauseSupplement (182) | Y | N | N |
| 暂停商品 (pause products) | `structPauseProduct` | Y | N | N |
| 暂停广告活动 (pause campaigns) | `structPauseCampaign` / pauseCampaign (20) | Y | N | N |

## The things that trip agents up

- **SD supports only two** - `budgetRedistribute` + `targetHarvest`. Nothing else on SD.
- **广告位调价 (placement bids) = SP only.** Enable it via
  `aiActionSettings.bidAdPlaceStatus = 1`. **For SD and SB: skip it - do not send
  `bidAdPlaceStatus`** (unsupported; it would be silently dropped).
- **Not available for SB (SP-only)**: 广告位调价, B2B调价, 定向暂停, 暂停商品, 暂停广告活动.
  If the user asks for any of these on an SB group, say it's SP-only and don't send it.
- **SB 分时调价 is Rule-only (no AI mode).** For SB bid daypart, you must set
  `aiAutomation.bidDaypartStatus = 1` (Rule mode) — AI mode is not supported for
  this capability on SB. On SP, AI mode (`bidDaypartStatus = 0` in aiAutomation)
  is supported.
- **`noRule` capabilities** (`budgetRedistribute`, `bidAmazonBusiness`) do **not**
  support Rule mode — they have **no** `aiAutomation` mode field. Control them
  only via `aiActionSettings.xxxStatus` = `0`/`1`. Do not send any `aiAutomation`
  field for these.

## Unsupported switches for the ad type - don't send them

If the user asks for an action space that isn't supported for their ad type (per the
table), say it isn't available and **don't send it**. Per the 2026-08-14 backend spec a
cross-type field should be **rejected with an error**; but that validation isn't fully
implemented yet, so today it may instead be **silently ignored** (call looks successful,
nothing changes). Either way the result is wrong - don't send it.
