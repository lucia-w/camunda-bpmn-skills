# DMN Integration Guide

Best practices for connecting DMN decision tables to Camunda BPMN business rule tasks.

## The Wiring Contract

Every `<businessRuleTask>` that calls a DMN decision has an implicit contract:

1. The task must supply every variable the DMN input expressions reference (via `zeebe:input`)
2. The task must capture every DMN output column the process cares about (via `zeebe:output` or via `resultVariable`)
3. The `resultVariable` must be non-empty and consistently named

Breaking any of these produces a silent wrong result or a runtime exception.

## Minimal Correct Wiring

```xml
<businessRuleTask id="checkInsuranceRebate" name="Check Insurance Rebate"
  zeebe:decisionId="insuranceRebateEligibility"
  zeebe:resultVariable="rebateDecision">
  <extensionElements>
    <!-- Supply everything the DMN input expressions reference -->
    <zeebe:input source="= hasInsuranceRebateTransaction" target="hasInsuranceRebateTransaction" />
    <zeebe:input source="= loanAmount" target="loanAmount" />

    <!-- Extract what the process needs from the result -->
    <zeebe:output source="= rebateDecision.rebateEligible" target="rebateEligible" />
    <zeebe:output source="= rebateDecision.rebateAmount" target="rebateAmount" />
  </extensionElements>
</businessRuleTask>
```

## `resultVariable` Structure

The raw DMN output is always stored as a map (or list of maps for multi-hit policies) in `resultVariable`. To use an individual output column downstream, you must either:

- Extract it with `zeebe:output`: `source="= rebateDecision.outputColumnName"`
- Or reference it inline in FEEL expressions: `= rebateDecision.rebateEligible`

Do not reference `resultVariable` columns directly as top-level variables unless you extracted them — they are nested under the result variable name.

## Hit Policy Reference

| Hit Policy | Result shape | Notes |
|------------|-------------|-------|
| `UNIQUE` | Single map or `null` | Most common — exactly one rule matches or none |
| `FIRST` | Single map | Returns first matching rule in order |
| `COLLECT` | List of maps | Consumer must handle a list |
| `RULE ORDER` | List of maps | Like COLLECT but ordered by rule number |
| `ANY` | Single map | All matching rules must produce the same output |

When the hit policy is `COLLECT` or `RULE ORDER`, downstream consumers must treat the result as a list. Using `= rebateDecision.rebateEligible` on a COLLECT result will fail — use `= rebateDecision[1].rebateEligible` or iterate with `for`.

## DMN Input Expression Tips

DMN input expressions reference variables by name. The name in the expression must **exactly** match the variable name in the Zeebe context at evaluation time — which means matching the `target` of the `zeebe:input` mapping, not the `source`.

```xml
<!-- zeebe:input target="hasInsuranceRebateTransaction" -->
<!-- DMN input expression: hasInsuranceRebateTransaction -->
<!-- These must match exactly — casing included -->
```

A common mistake: the BPMN variable is `hasInsuranceRebateTransaction` (camelCase) but the DMN input expression uses `has_insurance_rebate_transaction` (snake_case). The DMN receives `null` and silently evaluates against it.

## Validating Wiring

Run `/dmn-consistency-check` to automatically verify that every DMN input column has a corresponding `zeebe:input` and every DMN output column is either extracted or intentionally discarded.

## Testing DMN Decisions

Test the DMN in isolation first using Camunda's DMN simulator or the Camunda Modeler's built-in decision table tester. Then test the full task wiring in a process instance test to confirm the `zeebe:input` and `zeebe:output` mappings produce the expected top-level variables.
