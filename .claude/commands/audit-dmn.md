---
description: Audit a Camunda 8 DMN decision table for correctness and completeness
allowed-tools: Read, Bash, Glob, Grep
argument-hint: <path/to/decisions.dmn>
---

Audit the Camunda 8 DMN file at `$ARGUMENTS` for correctness issues that would produce wrong results or runtime errors. If no path is given, discover all `*.dmn` files in the project with `find . -name "*.dmn"` and audit each one.

## Steps

1. **Read the file** — parse the raw XML (DMN 1.3 namespace: `https://www.omg.org/spec/DMN/20191111/MODEL/`). List every `<decision>` element found.

2. **Check decision structure**
   - Each `<decision>` must have a unique, non-empty `id` and a human-readable `name`.
   - Decisions with a `<decisionTable>` child are rule-based; decisions with a `<literalExpression>` child are expression-based; check both.
   - Flag any `<decision>` element that has no inner logic at all (empty body).

3. **Check hit policy**
   - Valid hit policies: `UNIQUE`, `FIRST`, `RULE ORDER`, `COLLECT`, `ANY`, `OUTPUT ORDER`, `PRIORITY`.
   - If `hitPolicy` is absent, the default is `UNIQUE` — note this explicitly.
   - `COLLECT` may have an aggregation attribute (`SUM`, `MIN`, `MAX`, `COUNT`); verify the output column type is numeric when aggregation is used.
   - `PRIORITY` and `OUTPUT ORDER` require the output column to define an `outputValues` list; flag if missing.
   - With `UNIQUE` and `ANY`, overlapping rules that return different outputs are logic bugs — flag if detectable from a static scan (e.g., identical input entries in two rules).

4. **Check input columns (`<input>`)**
   - Every `<input>` must have a non-empty `<inputExpression>` with an `<text>` child that contains a valid FEEL path or expression (e.g., `applicant.age`, `order.totalAmount`).
   - `typeRef` should be set (`string`, `integer`, `long`, `double`, `boolean`, `date`, `time`, `dateTime`) — flag missing type refs as they suppress type checking.
   - `inputValues` (allowed values list) is optional but if present, every non-empty input entry in the rules should be a subset of that list — flag mismatches.

5. **Check output columns (`<output>`)**
   - Every `<output>` must have a non-empty `name` attribute (used as the output variable name).
   - `typeRef` should be set for the same reasons as inputs.
   - `outputValues` is required for `PRIORITY` and `OUTPUT ORDER` hit policies.
   - Duplicate output `name` values within a single decision table produce a map with collision — flag this.

6. **Check rules (`<rule>`)**
   - Each `<rule>` must have the same number of `<inputEntry>` elements as there are `<input>` columns, and the same number of `<outputEntry>` elements as `<output>` columns. Flag count mismatches.
   - An `<inputEntry>` with an empty `<text>` means "match any" — this is valid but note it when many inputs are empty (may indicate an overly broad catch-all rule that shadows other rules in `FIRST` tables).
   - An `<outputEntry>` with an empty `<text>` outputs `null` — flag this for non-COLLECT tables where null output is likely unintentional.
   - Check all `<inputEntry>` FEEL expressions for obvious syntax errors (unmatched quotes, unmatched brackets, invalid operators). See FEEL syntax reference in step 8.
   - Check all `<outputEntry>` FEEL expressions similarly.
   - Rules that are identical (same input entries, same output entries) are duplicates — flag them by rule number.

7. **Check required decision coverage**
   - For `UNIQUE` hit policies, if every input column has `inputValues` defined, perform a lightweight exhaustiveness check: count whether the combination of input value sets is fully covered by the rule set. Flag if there are obvious gaps (e.g., boolean column with only `true` rules but no `false` rule).
   - Note gaps for non-exhaustive tables — an uncovered input row will return `null` (or throw in strict mode).

8. **Check FEEL expression syntax in inputs and outputs**
   Use these heuristics to flag likely syntax errors:
   - Strings must be in double quotes: `"pending"` not `pending`.
   - Ranges use `..` inside `[` or `(`: `[1..10]`, `(0..100)`.
   - Negation: `not("cancelled")` — not `!= "cancelled"` (which is not FEEL).
   - Date literals: `date("2025-01-01")` — not bare `2025-01-01`.
   - Duration: `duration("P1D")`.
   - Function calls: `contains(text, "abc")`, `list contains(statuses, "ACTIVE")`.
   - Comparison operators: `< 100`, `>= 0`, `= "value"` — note `=` not `==`.
   - Disjunctions: `"a", "b", "c"` (comma-separated in an input entry matches any of them).

9. **Check DRD references (Decision Requirements Diagram)**
   - If the DMN file contains `<informationRequirement>` elements, verify that every `requiredDecision href` ID resolves to a `<decision>` that exists in the same file (or note if it may be in another file).
   - Circular dependencies between decisions will cause a deployment error — detect any cycles in the dependency graph.

10. **Cross-reference with BPMN (if BPMN files exist)**
    - Search for `*.bpmn` files in the project.
    - For each `businessRuleTask` with a `zeebe:calledDecision` extension, verify:
      - `decisionId` matches a `<decision id>` in a DMN file.
      - `resultVariable` is non-empty.
    - Flag BPMN references to decision IDs that don't exist in any DMN file found.

## Output format

```
## DMN Audit: <filename>

### Decision: "<decision-name>" (id: <decision-id>)

**Hit Policy:** <UNIQUE | FIRST | …>  
**Inputs:** <N>  **Outputs:** <M>  **Rules:** <K>

#### ✅ Passed
- Input column "applicant.age" has typeRef integer ✓
- ...

#### ⚠️ Warnings
- [COVERAGE] No rule covers creditScore > 800 — will return null.
- [NULL-OUTPUT] Rule 3, output "result": empty outputEntry will produce null.
- ...

#### ❌ Errors
- [FEEL-SYNTAX] Rule 5, input "status": `pending` is not quoted — should be `"pending"`.
- [COLUMN-MISMATCH] Rule 7 has 2 inputEntry elements but table has 3 input columns.
- ...

---
### 📋 Summary
| Decision | Errors | Warnings |
|---|---|---|
| <name> | N | M |

**Overall verdict: READY / NEEDS FIXES**
```
