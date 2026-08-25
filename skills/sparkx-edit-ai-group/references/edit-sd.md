# Edit SD managed groups - `edit_sd_ai_managed_group`

Updates one or more **SD** groups. Required: `profileId` and `ids` (array of
aiGroupIds, **max 20**).

## Two modes

`edit_sd_ai_managed_group` behaves differently depending on whether `operation` is sent:

- **Legacy full edit (`operation` omitted).** Every non-null field you send is applied
  to all groups in `ids`. Use this only for changing several settings at once. **For a
  single change (e.g. AI on/off) prefer the batch form below** - in pre, a Legacy
  full-edit for a plain SD status change returned a misleading `only supports SD groups`
  error, while the batch `{operation:"STATUS", status:1}` succeeded.
- **Batch operation (`operation` provided).** One operation per request - SD is
  **flat/top-level**: put `operation` and its params directly alongside `profileId`/`ids`
  (no `request` wrapper, no `batchParams`), e.g. `{profileId, ids, operation:"BUDGET",
  budgetType:3, budgetRatio:20}`. Operation values are **UPPER_SNAKE_CASE** (prod-confirmed
  2026-08-13); verify the exact enum in the tool schema. See
  [`batch.md`](batch.md) - **the "SD" section** - for the shape, id<->profile validation,
  and three-state (`success` / `partial_failure` / `failed`) result handling. (SP/SB uses
  a different, `request`-wrapped shape.)

The rest of this file documents the fields (used by both modes).

## Type fields decide which value field is valid - never mix

`acosType` / `roasType` / `budgetType` are "mode" selectors. The mode dictates exactly
which companion field to send; sending the wrong one, or both, fails. The tool does not
guard this (MCP bypass) - you must.

**ACOS** (`acosType`):

| `acosType` | Meaning | Send | Do NOT send |
|---|---|---|---|
| `1` | specific value | `acos` (required) | `acosRatio` |
| `2` | recommended | `acosStatus=1` (apply recommended); no `acos` needed | `acos`, `acosRatio` |
| `3` | ratio adjust | `acosRatio` (required) | `acos` |

**ROAS** (`roasType`): same shape - `1`->`roas` (no `roasRatio`); `2`->recommended;
`3`->`roasRatio` (no `roas`).

**Budget** (`budgetType`):

| `budgetType` | Meaning | Send | Do NOT send |
|---|---|---|---|
| `1` | total budget | `budget` (required) | `budgetRatio` |
| `2` | daily budget | `budget` (required) | `budgetRatio` |
| `3` | ratio adjust | `budgetRatio` (required) | `budget` |

> **`budgetType=3` is a known-risky value** (can hang the downstream ai-center service).
> Avoid it unless the user specifically needs ratio mode; if you do use it, send
> `budgetRatio` and no `budget`.

## Other coupled fields

- **Dynamic budget** (`budgetDynamicStatus=1`) -> also `numType` (`1`=percentage,
  `2`=fixed) and `num` (the value).
- A target ACOS/ROAS value must respect the **x100 scale** (25% -> `25`).

## Other editable fields

| Field | Values |
|---|---|
| `status` | `0`=off, `1`=on |
| `optimizeType` | `1`=drive growth, `2`=maintain stability, `3`=boost volume. Values outside 1-3 are rejected (`Parameter 'optimizeType' must be 1 (drive growth), 2 (maintain stability), or 3 (boost volume)`). Note the tool's own inline description still lists only 1 and 2 — 3 is accepted; the validation is authoritative |
| `targetHarvestStatus` | `0`=off, `1`=on, `2`=on with exact negation |
| `budgetRedistributeStatus` | `0`/`1` |
| `aiPersonality` | `1`-`5` |
| `campaignNameSign` | `0`/`1` |
| `campaignNameRecoveryType` | when turning the label off: `1`=keep current name, `2`=restore original name |
| `remark` | free text, max 200 chars |

## Example - Legacy multi-group edit: adjust ACOS by +10% and turn AI on

```json
{
  "profileId": 3721212165742,
  "ids": [826117, 826120],
  "acosType": 3,
  "acosRatio": 10,
  "status": 1
}
```

Then verify each id via `get_entity_metadata(profileIds=[...], entity='aiGroup', ...)`.

> SD-only. For SP/SB use `save_sp_sb_ai_managed_group` (edit mode) - its `acos` is a
> plain number with **no** `acosType` machinery.
