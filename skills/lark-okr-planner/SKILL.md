---
name: lark-okr-planner
description: Draft and create OKRs (objectives, key results, weights, alignment) in Lark/Feishu for any team or role (engineering, sales, marketing, operations, product, support, etc.) with lark-cli, including the rich-text content format that avoids objectives saved as "*". Use when planning quarterly or other-cycle OKRs, turning goals into measurable key results, or creating and editing OKRs in Lark.
---

# Lark OKR Planner

Turn goals for any team or role into well-formed OKRs, then create them in Lark with `lark-cli`.

## When to Use

- Drafting OKRs for a team or individual for any cycle (quarter, half-year, year)
- Rewriting vague goals into measurable key results
- Creating or editing objectives and key results in Lark, including weights
- Checking how an OKR set aligns to a parent objective

For the full, current `lark-cli okr` command set (shortcuts such as `+cycle-list`, `+create`, `+batch-create`, `+patch`, `+weight`, progress records, comments), run `lark-cli okr --help`. This skill adds the planning method and the raw-API flow with its pitfalls.

## Step 1: Draft before touching Lark

Get these from the user; do not invent them:

- Owner (person or team) and cycle (for example `2026-Q2`)
- The goals or priorities for the cycle, and the parent objective if aligning
- Current baselines and target values for each metric

Where a number is missing, leave a visible placeholder such as `[target]` and ask. Never fill in targets for the user.

### Writing rules

| Part | Rule |
| :--- | :--- |
| Objective | Qualitative, inspiring, owned by one team, achievable within the cycle. Describes the outcome, not the activity. Typically 2–5 per cycle. |
| Key Result | Measurable outcome with a baseline, a target, and a metric. Verbs like "ship" or "launch" alone are tasks, not KRs. Typically 2–5 per objective. |
| Weight | Share of the parent's importance. Weights of sibling objectives (and of KRs under one objective) should sum to 100%. |
| Alignment | Link an objective to the parent objective or KR it supports. Top-level objectives need no parent. |

### Examples by role

The bracketed values are placeholders. Replace them with the team's real baselines and targets.

| Role | Objective | Example key result shape |
| :--- | :--- | :--- |
| Engineering | Make the platform reliable | Reduce `[metric]` from `[baseline]` to `[target]` |
| Sales | Grow qualified pipeline | Increase `[pipeline metric]` from `[baseline]` to `[target]` |
| Marketing | Build demand in `[segment]` | Reach `[target]` `[conversion metric]` by `[date]` |
| Operations | Shorten fulfilment | Cut median `[process]` time from `[baseline]` to `[target]` |
| Product | Improve activation | Raise `[activation rate]` from `[baseline]` to `[target]` |
| Support | Resolve issues faster | Keep `[satisfaction metric]` at or above `[target]` |

Show the draft to the user and get approval before creating anything in Lark.

## Step 2: Find the cycle

```bash
lark-cli okr +cycle-list --format json
```

Pick the cycle that matches the requested period (use `start_time` / `end_time` in the output) and note its `id`. For another user's cycles, check `lark-cli okr +cycle-list --help` for the user flag. Example filter by start month:

```bash
lark-cli okr +cycle-list --format json | jq '.data.cycles[] | select(.start_time | startswith("YYYY-MM")) | .id'
```

## Step 3: Create objectives (empty first)

Create with empty data, then patch in the content and weight:

```bash
# Create objective
lark-cli okr cycle.objectives create \
  --params '{"cycle_id":"CYCLE_ID"}' \
  --data '{}' \
  --format json | jq -r '.data.objective_id'

# Update objective with content and weight (0.4 = 40%)
lark-cli okr objectives patch \
  --params '{"objective_id":"OBJ_ID"}' \
  --data '{
    "content": {
      "blocks": [
        {
          "block_element_type": "paragraph",
          "paragraph": {
            "elements": [
              {
                "paragraph_element_type": "textRun",
                "text_run": {
                  "text": "Objective text here"
                }
              }
            ]
          }
        }
      ]
    },
    "weight": 0.4
  }' \
  --format json
```

## Step 4: Create key results

For each objective:

```bash
# Create key result
lark-cli okr objective.key_results create \
  --params '{"objective_id":"OBJ_ID"}' \
  --data '{}' \
  --format json | jq -r '.data.key_result_id'

# Update key result with content and weight
lark-cli okr key_results patch \
  --params '{"key_result_id":"KR_ID"}' \
  --data '{
    "content": {
      "blocks": [
        {
          "block_element_type": "paragraph",
          "paragraph": {
            "elements": [
              {
                "paragraph_element_type": "textRun",
                "text_run": {
                  "text": "Key result text here"
                }
              }
            ]
          }
        }
      ]
    },
    "weight": 0.5
  }' \
  --format json
```

## Step 5: Alignment

Alignment links an objective to a parent objective or KR. Manage it through the `alignments` resource of `lark-cli okr` (see `lark-cli okr --help` and the upstream `lark-okr` skill's alignment guide). Confirm the parent's ID with the user before linking, and read the result back after creating it.

## Step 6: Verify

Read the cycle back (`lark-cli okr +cycle-detail --help` shows the options) and check:

- Every objective and KR text shows as written, not as `*`
- Sibling weights sum to the intended total (the API may normalize them)
- Alignment links point at the intended parents

Report what was created and anything left as a placeholder.

## Authorization

Required scopes:
- `okr:okr.period:readonly` - Read OKR cycles
- `okr:okr.content:writeonly` - Create/update objectives and key results
- `okr:okr.setting:read` - Read OKR settings

Request via:
```bash
lark-cli auth login --scope "okr:okr.period:readonly okr:okr.content:writeonly okr:okr.setting:read" --no-wait --json
```

## ContentBlock Text Format (CRITICAL)

**WRONG** - This will save as "*":
```json
{
  "elements": [
    {
      "type": "text",
      "text": "My Text"
    }
  ]
}
```

**CORRECT** - Use `textRun` with `paragraph_element_type`:
```json
{
  "elements": [
    {
      "paragraph_element_type": "textRun",
      "text_run": {
        "text": "My Text"
      }
    }
  ]
}
```

Key differences:
- Element type must be `paragraph_element_type: "textRun"` (not `type: "text"`)
- Content wrapped in `text_run` object
- Text goes in `text_run.text` field

## Learnings

1. **Empty data works for creation**: Use `--data '{}'` to create objectives/KRs without content, then patch to add text. This avoids parameter validation issues.

2. **ContentBlock requires proper structure**: The Lark OKR API uses a rich text format where text must be wrapped in `textRun` elements within `paragraph` blocks. Using the wrong structure will display as asterisks.

3. **Weight normalization**: When you set weights for objectives/KRs, the API may normalize them (e.g., 0.4 + 0.35 + 0.25 = 1.0). Verify final weights after creation.

4. **Patch after create**: Create empty objects first, then patch with content. This is more reliable than trying to include everything in the initial create request.

5. **Score is not progress**: Set a score only when the user explicitly asks for one. Progress or completion updates go through progress records or the indicator, not the score.
