# DMN ↔ BPMN Consistency Check

Verifies that every DMN decision table is correctly wired to its calling business rule task in BPMN: inputs align, outputs are captured, and no variable is silently dropped between the two.

## Usage

```
/dmn-consistency-check [bpmn-file] [dmn-file]
```

If no files are given, scan all `*.bpmn` and `*.dmn` files in the current directory tree and match them by decision reference.

## What This Checks

### For Each Business Rule Task (`<businessRuleTask>`)

1. **Decision reference exists** — the `decisionRef` value matches a `decision/@id` in a DMN file on disk.

2. **DMN inputs → BPMN zeebe:inputs**
   For every `<input>` column in the decision table (`<inputExpression>`):
   - The variable name referenced in the expression must have a corresponding `zeebe:input` on the task, OR must already be a process-level variable guaranteed to be in scope.
   - Flag any DMN input expression that references a variable with no `zeebe:input` mapping.

3. **BPMN zeebe:inputs → DMN inputs**
   For every `zeebe:input` on the task:
   - There should be a DMN input column that consumes it, OR it is a context variable the DMN rule engine uses implicitly.
   - Flag `zeebe:input` entries that map to nothing in the DMN (dead mappings).

4. **DMN outputs → BPMN zeebe:outputs**
   For every `<output>` column in the decision table:
   - The output name must appear as a key in the `resultVariable` map, OR there must be an explicit `zeebe:output` that extracts it.
   - Flag any DMN output column whose value is never captured.

5. **resultVariable is set and used**
   - `resultVariable` must be non-empty.
   - The variable named in `resultVariable` must appear in at least one downstream `zeebe:input` or be referenced in a gateway condition or another task's input — otherwise the output is produced but never consumed.

6. **Hit policy compatibility**
   - If the DMN hit policy is `COLLECT` or `RULE ORDER`, the consumer must handle a list, not a scalar. Check that downstream usage treats it as a list.
   - If the hit policy is `UNIQUE` or `FIRST`, downstream code should not assume a list.

### Reporting Format

```
## Decision: "Discount Eligibility" (discountEligibility.dmn)
Called by: "Calculate Discount" task in order-fulfillment.bpmn

DMN Inputs
- ❌ `hasLoyaltyMembership` — referenced in DMN input expression but no zeebe:input mapping found on task
- ✅ `orderAmount` — mapped via zeebe:input from `= orderAmount`

DMN Outputs
- ✅ `discountApplied` — captured in resultVariable `decisionResult`, extracted via zeebe:output
- ⚠️  `discountRate` — present in DMN output column but no zeebe:output extracts it; confirm intentional

Hit Policy: UNIQUE — downstream usage OK (scalar access)
```

## Instructions

1. Find all `<businessRuleTask>` elements across BPMN files.
2. For each, resolve the `decisionRef` to a DMN file.
3. Parse the DMN `<decisionTable>` for input and output columns.
4. Cross-reference against the task's `zeebe:input`/`zeebe:output`/`resultVariable`.
5. Report per-decision findings using the format above.
6. Summarise: total decisions checked, total mismatches, total warnings.
