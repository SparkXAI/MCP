# Reading Automation Rules

Use this guide when the user asks which automation rules are enabled or asks you to explain a
Rule-mode configuration. It translates the MCP response into business meaning without claiming
that the read tool exposes the complete standalone automation-template system.

## 1. Choose the Read Surface

| User question | Query | What it returns | What it does not return |
|---|---|---|---|
| Which rule types are enabled on these campaigns? | `entity: "automationRule"`, filtered by `amazonCampaignId` | Campaign ID plus `enabledRuleTypes` and localized `enabledRuleNames` | Template ID/name, conditions, actions, frequency, applicable objects, account template list |
| Explain this managed group's Rule-mode setup | `entity: "aiGroup"`, filtered by `aiGroupId` | Effective action-space switches and retained `aiAutomation.{ruleType}` configuration | Standalone account-level automation templates |
| Explain a managed-group schedule's embedded rules | `entity: "aiGroup_schedule"`, one profile and one group | Per-schedule effective configuration when the backend returns it | Standalone templates; missing inherited group objectives must be read from the parent group |
| List or inspect standalone automation templates | Not supported by the current metadata entities | - | The Help Center template library and full standalone template configuration |
| Inspect product inventory rules | Not supported by these campaign/managed-group surfaces | - | SKU/product rule associations and inventory-rule configuration |

The Help Center overview still says seven templates, while its current body also documents newer
Budget Rules, Placement Rules, and Campaign activation/pausing. Do not infer coverage from that
count; use the returned rule codes and the capability table below.

## 2. Determine Off, AI, or Rule Mode

Always pair the action-space switch in `aiActionSettings` with the rule entry in
`aiAutomation`. Do not classify mode from either object alone.

| ruleType | Managed-group action space | Paired effective switch |
|---:|---|---|
| `2` | Bid dayparting | `bidOptimization.bidDaypartStatus` |
| `4` | Harvest targets / add search terms | `targetOptimization.targetHarvestStatus` |
| `5` | Add negative targets | `targetOptimization.negativeTargetStatus` |
| `13` | Budget dayparting | `budgetOptimization.budgetDaypartStatus` |
| `17` | Budget rules / boost budget | `budgetOptimization.budgetDynamicStatus` |
| `19` | Placement bid adjustment | `bidOptimization.bidAdPlaceStatus` |
| `20` | Pause campaigns and ad groups | `structOptimization.structPauseCampaignStatus` |
| `181` | Optimize base bids | `bidOptimization.bidPerformanceStatus` |
| `182` | Pause targets | `targetOptimization.targetPausedAddStatus` |

Decision procedure:

1. If the paired switch is `0`, report the action space as off. Any dependent fields omitted by
   the effective-config projection are expected.
2. If the switch is on and `aiAutomation.{ruleType}.status=1` is present, report Rule mode and
   explain the retained configuration.
3. If the switch is on and no retained entry exists for that rule type, report AI mode. AI-mode
   placeholders are removed by the server.
4. If the switch is missing or malformed, say the effective mode cannot be determined from this
   row. Do not turn an empty `aiAutomation` into "all AI" or "all off".

`targetPausedAddStatus=2` is an AI-only "pause plus supplement" option. In Rule mode the response
folds it to effective value `1`; describe rule `182` as pausing targets, not supplementing them.

## 3. Read the Rule Tree Safely

- Prefer `ruleTypeText`, `statusText`, and leaf-level `...Text` / `...Formatted` fields. Keep the
  raw value beside the label when precision matters.
- Treat rule arrays as ordered. The Help Center states that when several strategies match, only
  the highest-priority strategy runs; the returned order is therefore meaningful and must not be
  sorted or deduplicated.
- Different condition groups are OR; conditions inside one group are AND. When the raw response
  exposes only connectors such as `operation`/`operationText`, preserve their order and wording
  instead of flattening every item into AND.
- Read `day` together with `exceptDay`: "past 7 days, excluding the latest 2" means the earlier
  five-day portion of that lookback. Do not describe it as all seven days.
- `todayPerformance` means the current store day from 00:00 through the execution time.
  `historyPerformance` uses the configured historical lookback.
- Unknown paths and values are intentionally passed through. If a leaf has no confirmed Text
  companion, show the raw key/value and label it as an unconfirmed code; never borrow a label from
  another rule type. The same raw token can have different meanings in different rules.

## 4. Rule-Specific Meaning

### `2` - Bid dayparting

- Primary payload: `updateBid`, with `templateDateType` when present.
- Matrix values are bid coefficients/multipliers, not percentage-point increases. A base bid of
  5 with coefficient 2 produces an effective bid of 10.
- The rule executes at the start of each configured hour. Use `updateBidFormatted` for matrix
  dimensions/coordinates when returned, but retain the raw matrix values.

### `13` - Budget dayparting

- Primary payload: the hourly budget matrix in `condition`; it may also carry
  `templateDateType`, misspelled upstream field `excuteDays`, and budget bounds.
- Business operations can be fixed budget, increase/decrease by percentage, or increase/decrease
  by amount. Decode a cell only when the response provides a confirmed Text value; otherwise show
  its raw operation/value.
- An unconfigured hour restores the campaign's current daily budget in SparkX AI. This reset behavior
  is the key difference from rule `17` scheduled budget changes.

### `17` - Budget rules / budget by performance

- `condition.setBudgetOption` is the scheduled-budget portion. A value set at one time continues
  until the next scheduled change; blank hours do not automatically restore the campaign budget.
- `condition.todayPerformance` is the current-day performance portion. It can increase, decrease,
  or set daily budget when conditions match. Ordered strategies are priority order.
- Budget utilization means current-day spend divided by the latest daily budget at execution
  time. The Help Center describes checks every 30 minutes inside configured execution windows.
- `autoModifySwitch` / `autoModifySetting` are optional budget callback/reset configuration. The
  server currently passes these raw; residual fields do not prove the callback is enabled.

### `19` - Placement rules

- Conditions live under `condition.historyPerformance`; `activeModel` describes the applicable
  ad type/object when present.
- A condition can target one placement, while its action array can adjust one or all placements.
  Keep `placementText` and each action's `placementText` separate.
- Increase/decrease is relative to the current placement adjustment: current 55% increased by
  10% is calculated as `55% + 55% * 10%` (subject to configured max/min). A fixed action sets the
  placement adjustment directly.
- The Help Center supports daily/weekly/monthly execution at a chosen whole hour. If the returned
  time fields have no Text/Formatted companion, report their raw values without assuming units.

### `20` - Campaign activation/pausing (new)

- `condition.timedSet` contains scheduled campaign state changes; schedules may be daily, weekly,
  or non-recurring. Scheduled actions can activate or pause.
- `condition.historyPerformance` contains historical-performance strategies. The Help Center's
  current version only performs **pause campaign** for this branch.
- If a campaign is already paused at evaluation time, the performance strategy does not continue.
  Keep scheduled state actions distinct from the rule's top-level `status` mode field.
- Deep rule-20 leaves are currently raw passthrough. Do not translate `setStatus` with the
  top-level Rule/AI status vocabulary.

### `181` - Optimize base bids

- Conditions live under `condition.historyPerformance`. Actions can increase, decrease, or set
  base bids subject to the returned value/bounds configuration.
- Use the action's `actionTypeText`, `baseTypeText`, and bound/switch Text fields when present.
  Use `timePeriodTypeText` and `timePeriod.hourFormatted` for the execution schedule.

### `182` - Pause targets

- Conditions live under `condition.historyPerformance`; the Rule-mode action is pause targeting.
- Use returned condition and time-period Text fields. Do not describe target supplementation in
  Rule mode; that behavior belongs to the AI-only value folded out by the effective projection.

### `4` / `5` - Harvest search terms / add negatives

- These use a legacy raw structure: top-level `condition` arrays plus rule-specific `action`,
  `isSelf`, `campaignAd`, and matching settings. Their deep read-path enums are not fully closed,
  so preserve raw codes unless a Text field is present.
- Help Center meaning: rule `4` adds qualifying search terms as keyword/product targets with a
  configured match method and bid; rule `5` adds qualifying search terms as exact/phrase negative
  targets.
- Standalone rules evaluate on selected weekdays around 03:00 in the store timezone. Do not apply
  that schedule to a managed-group response unless its returned configuration supports it.

## 5. Coverage by Surface

| Rule/code | Standalone campaign enabled-type lookup | Managed-group `aiAutomation` config |
|---|---|---|
| `2`, `4`, `5`, `13`, `17`, `19`, `20` | Yes | Yes, when the paired action space is in Rule mode |
| `3`, `6`, `7`, `8` | Legacy standalone types may appear | No current managed-group mapping |
| `15`, `18` | Targeting-rule types may appear | No current managed-group mapping |
| `181`, `182` | May be returned by the shared type-name provider, but they are managed-group action-space rules | Yes |
| Product inventory rules | Not represented by this campaign lookup | No |

## 6. Recommended Explanation Shape

For each rule, report:

1. Rule name/code and effective mode.
2. Applicable action space/object, only when exposed.
3. Conditions, preserving group connectors and priority order.
4. Action and bounds.
5. Data period and execution schedule.
6. Any raw/unconfirmed fields and the read-surface limitation.

Never claim that a managed-group read is the account's reusable automation template, and never
claim that an enabled-type lookup contains the rule's actual configuration.
