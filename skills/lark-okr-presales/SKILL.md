---
name: lark-okr-presales
description: Create presales OKRs in Lark with objectives, key results, and proper text formatting for Q2-style quarterly planning.
---

# Lark OKR Presales Skill

Create comprehensive presales OKRs (Objectives and Key Results) in Lark with proper formatting and text content.

## When to Use

- Creating OKRs for presales roles focused on pipeline and deal closure
- Need to set up quarterly presales targets aligned with sales goals
- Building OKR framework with 3 objectives and 6 key results

## OKR Structure for Presales

Standard presales OKR framework (Q2 2026 example). The weights and target numbers below are **illustrative placeholders**; replace them with your own targets before creating OKRs:

**Objective 1: Expand Deal Pipeline (40% weight)**
- Increase qualified leads by 40%
- Generate 50+ new high-value prospects

**Objective 2: Accelerate Deal Closure (35% weight)**
- Reduce average sales cycle by 30%
- Achieve 75% deal win rate for qualified opportunities

**Objective 3: Build Strategic Customer Relationships (25% weight)**
- Conduct 30+ deep discovery/demo sessions
- Achieve 90%+ customer satisfaction score in presales engagements

## Process

### 1. Get Cycle ID

First, identify the OKR cycle for your quarter:

```bash
lark-cli okr +cycle-list --format json | jq '.data.cycles[] | select(.start_time | startswith("2026-04")) | .id'
```

### 2. Create Objectives (Empty First)

Create objectives with empty data, then update with content:

```bash
# Create objective
lark-cli okr cycle.objectives create \
  --params '{"cycle_id":"CYCLE_ID"}' \
  --data '{}' \
  --format json | jq -r '.data.objective_id'

# Update objective with content and weight
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
                  "text": "Objective Name Here"
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

### 3. Create Key Results

For each objective, create key results:

```bash
# Create key result
lark-cli okr objective.key_results create \
  --params '{"objective_id":"OBJ_ID"}' \
  --data '{}' \
  --format json | jq -r '.data.key_result_id'

# Update key result with content
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
                  "text": "Key Result Text Here"
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

## Authorization Requirements

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
