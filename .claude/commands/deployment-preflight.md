# Deployment Preflight

Runs all review checks in sequence before a Camunda process deployment. Aggregates results from BPMN, DMN, event, expression, worker, form, and versioning checks into a single go/no-go report.

## Usage

```
/deployment-preflight [file-or-directory]
```

Run this in the root of a process module before raising a PR or deploying to any environment.

## What This Runs

Executes the following checks in order:

1. **BPMN Step Review** — service task mappings, gateway defaults, retry config
2. **DMN Consistency Check** — DMN input/output alignment with business rule tasks
3. **Call Activity Audit** — sub-process references, input/output scope isolation
4. **Error Handling Review** — boundary events, error codes, unhandled failure paths
5. **Message Event Audit** — message references, correlation keys, payload assumptions
6. **Timer Event Audit** — timer formats, FEEL timer variables, timeout paths
7. **Multi-Instance Review** — collection wiring, item scope, output aggregation
8. **FEEL Expression Review** — expression syntax, null safety, variable availability
9. **Worker Contract Audit** — job types, worker inputs/outputs, headers, error codes
10. **Form and User Task Review** — assignments, forms, form variables, manual paths
11. **Process Versioning Audit** — artifact IDs, call activity binding, decision references

## Go / No-Go Criteria

**Block deployment (❌ failures present):**
- Any missing `zeebe:input` or `zeebe:output` that will cause a null variable at runtime
- Any DMN input column with no matching task input mapping
- Any call activity referencing a non-existent process ID
- Any catch-all error boundary that swallows unexpected failures silently
- Any service task with retries=0 and no error boundary event
- Any message event with a missing or unsafe correlation key
- Any invalid timer expression or timeout path that strands the token
- Any parallel multi-instance activity that races on the same output variable
- Any FEEL expression that references an unavailable variable on an executable path
- Any worker job type, required input, output shape, or BPMN error code mismatch
- Any user task form reference or required form output missing from the deployment package
- Any duplicate process or decision ID in the deployment package

**Warn but do not block (⚠️ warnings present):**
- Retries set to non-default values without a comment
- Dead mappings (inputs mapped in but never consumed)
- `propagateAllChildVariables=true` or defaulted broad child propagation on any call activity
- Hit policy / consumer type mismatch
- Message or timer behavior that is valid but operationally undocumented
- Latest-version call activity or decision binding where pinning may be safer
- Optional form variables used downstream without explicit null handling

## Reporting Format

```
# Deployment Preflight — order-fulfillment.bpmn + discountEligibility.dmn
Run: 2024-01-15

## Results by Check

| Check                   | Files | ✅ Pass | ⚠️ Warn | ❌ Fail |
|-------------------------|-------|--------|--------|--------|
| BPMN Step Review        | 3     | 18     | 2      | 1      |
| DMN Consistency         | 2     | 6      | 0      | 1      |
| Call Activity Audit     | 3     | 4      | 1      | 0      |
| Error Handling Review   | 3     | 7      | 1      | 0      |
| Message Event Audit     | 3     | 5      | 1      | 0      |
| Timer Event Audit       | 3     | 2      | 0      | 0      |
| Multi-Instance Review   | 3     | 1      | 0      | 0      |
| FEEL Expression Review  | 4     | 21     | 2      | 1      |
| Worker Contract Audit   | 3     | 9      | 1      | 0      |
| Form/User Task Review   | 3     | 3      | 0      | 0      |
| Versioning Audit        | 4     | 8      | 1      | 0      |

## ❌ DEPLOY BLOCKED — 2 failures must be resolved

### Failure 1 (BPMN Step Review)
File: order-fulfillment.bpmn | Task: "Reserve Inventory"
Missing zeebe:output for `reservationId` — gateway condition `= reservationId != null` will throw at runtime

### Failure 2 (DMN Consistency)
File: order-fulfillment.bpmn | Task: "Calculate Discount"
DMN input `hasLoyaltyMembership` has no matching zeebe:input mapping

## ⚠️ Warnings (non-blocking)
- call-activity-audit: "Process Payment" has broad child variable propagation enabled
- bpmn-step-review: "Send Confirmation Email" has retries=5, confirm backoff is set
- error-handling-review: "Validate Order" catch-all boundary event — consider specific error code
```

## Instructions

1. Run each sub-check against all files in scope.
2. Collect all ❌ failures and ⚠️ warnings.
3. Print the summary table.
4. If any ❌ failures: print "DEPLOY BLOCKED" with each failure in detail.
5. If only ⚠️ warnings: print "DEPLOY WITH CAUTION" with warnings listed.
6. If all clear: print "READY TO DEPLOY" with the pass counts.
