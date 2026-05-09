# BPMN Common Anti-patterns

Patterns that pass schema validation but break at runtime or cause subtle process bugs.

---

## 1. Missing Gateway Default

**The bug:** An exclusive gateway has conditions on all outgoing flows, but at runtime none of them evaluate to true. Zeebe throws a "no outgoing sequence flow" exception and incidents the token.

```xml
<!-- BAD: no default, both conditions could be false -->
<exclusiveGateway id="loanDecision" />
<sequenceFlow sourceRef="loanDecision" targetRef="approve">
  <conditionExpression>= creditScore >= 700</conditionExpression>
</sequenceFlow>
<sequenceFlow sourceRef="loanDecision" targetRef="decline">
  <conditionExpression>= creditScore < 600</conditionExpression>
</sequenceFlow>
<!-- What happens when creditScore is 650? -->
```

**Fix:** Set the `default` attribute to one flow ID, or ensure conditions are exhaustive (use `>= 600` not `< 600`).

---

## 2. Implicit Variable Inheritance in Call Activities

**The bug:** A call activity relies on the parent's variables being available in the child without explicit `zeebe:input` mappings. Works by accident when `propagateAllParentVariables=true`, breaks when it is `false` or when the platform version changes.

**Fix:** Always map explicitly. Treat every call activity as a function call with explicit parameters.

---

## 3. `resultVariable` Left Empty on Business Rule Task

**The bug:** `zeebe:resultVariable` is not set. The DMN output is discarded. No error is thrown — the process continues with `null` results.

**Fix:** Always set `resultVariable` to a meaningful name. Always extract output columns via `zeebe:output`.

---

## 4. Variable Name Casing Mismatch (BPMN → DMN)

**The bug:** BPMN maps `hasInsuranceRebateTransaction` (camelCase). DMN input expression references `has_insurance_rebate_transaction` (snake_case). DMN receives `null`, evaluates against it, produces wrong output. No exception thrown.

**Fix:** Use consistent naming across BPMN variables and DMN input expressions. Run `/dmn-consistency-check` to catch these.

---

## 5. Output Variable Not Mapped Out of Call Activity

**The bug:** A sub-process produces a result variable (`vaultId`). The parent does not have a `zeebe:output` on the call activity. The parent reads `vaultId` downstream — gets `null`. No exception until the variable is actually used.

**Fix:** Always trace variables that cross call activity boundaries. Use `/variable-trace <varName>` to verify.

---

## 6. Catch-all Error Boundary

**The bug:** An `<errorEventDefinition>` with no `errorRef` catches everything, including programming errors, null pointers, and config mismatches. The process routes to the "known error" path when actually something unexpected broke.

**Fix:** Always specify a specific `errorCode`. If you genuinely want to catch all BPMN errors, document why.

---

## 7. Timer in Wrong Format

**The bug:** A timer event uses a human-readable string like `"10 seconds"` instead of ISO 8601. Zeebe silently uses a zero or default duration.

```xml
<!-- BAD -->
<timeDuration>10 seconds</timeDuration>

<!-- GOOD -->
<timeDuration>PT10S</timeDuration>
```

**Fix:** Always use ISO 8601. `PT` prefix = duration. `P` prefix = period. Examples: `PT30S`, `PT5M`, `P1D`, `P1W`.

---

## 8. Parallel Gateway with Unbalanced Branches

**The bug:** A parallel split (+) has 3 outgoing branches, but the joining parallel gateway expects 2 incoming flows. One branch never completes the join — the process hangs forever.

**Fix:** The split gateway and join gateway must have matching fan-out and fan-in counts. Use `/bpmn-step-review` to check for this.

---

## 9. Service Task Job Type Mismatch

**The bug:** BPMN declares job type `credit-score-fetch`. The worker subscribes to `creditScoreFetch` (camelCase vs kebab-case). The job sits in Zeebe forever — no worker picks it up, no immediate error.

**Fix:** Establish and enforce a naming convention for job types across all processes and workers. Kebab-case is conventional.

---

## 10. Reusing Variable Names Across Parallel Branches

**The bug:** Two parallel branches both write to a variable named `status`. Depending on which branch completes last, the parent sees either value — a race condition in the process definition.

**Fix:** Use unique variable names per branch (`approvalStatus`, `creditStatus`) and merge them intentionally after the join gateway.
