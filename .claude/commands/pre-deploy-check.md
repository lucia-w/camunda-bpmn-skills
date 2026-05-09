---
description: Comprehensive pre-deployment readiness check for all Camunda 8 BPMN and DMN files in the project
allowed-tools: Read, Bash, Glob, Grep
argument-hint: [path/to/directory or file]
---

Run a full pre-deployment readiness check on all Camunda 8 / Zeebe BPMN and DMN files in `$ARGUMENTS` (defaults to the entire project if no argument is given). This command combines all individual checks into a single pass and produces a deployment go/no-go verdict.

## Steps

### 1. Discover all process files

```bash
find ${ARGUMENTS:-.} \( -name "*.bpmn" -o -name "*.dmn" \) | sort
```

List every file found. If none are found, stop and report "No BPMN or DMN files found."

### 2. Validate each BPMN file — apply all checks from `/review-bpmn`

For every `*.bpmn` file:

#### 2a. Structural integrity
- Exactly one None Start Event per top-level process.
- At least one End Event per process.
- All SequenceFlow `sourceRef` and `targetRef` IDs resolve to existing elements.
- No unreachable elements (no incoming flow, not a start event).
- No dead-end elements (no outgoing flow, not an end event or waiting intermediate catch event).

#### 2b. Gateway logic
- Exclusive/Inclusive Gateways: all non-default outgoing flows have `conditionExpression`.
- Parallel Gateways: outgoing flows have no conditions.
- Event-Based Gateways: all targets are intermediate catch events.

#### 2c. Service tasks
- `zeebe:taskDefinition type` is non-empty for every service task.
- No duplicate `zeebe:header key` values per task.

#### 2d. Message events
- All `messageEventDefinition` reference a `<message name>` that is non-empty.
- Intermediate/boundary message catch events have a non-empty `zeebe:subscription correlationKey`.

#### 2e. Timer events
- `timeDuration`, `timeCycle`, `timeDate` are valid ISO 8601 or FEEL expressions.

#### 2f. Error events
- `errorEventDefinition` references an `<error errorCode>` that is non-empty.

#### 2g. Signal events
- `signalEventDefinition` references a `<signal name>` that is non-empty.

#### 2h. Call activities
- `zeebe:calledElement processId` is non-empty.

#### 2i. User tasks
- User tasks have a `zeebe:formDefinition` or `formKey`.

#### 2j. Variable mappings
- `zeebe:input` and `zeebe:output` elements have both `source` and `target`.
- No duplicate `target` names within the same mapping block.

#### 2k. Sub-processes
- Embedded sub-processes have their own start and end events.
- Event sub-processes have exactly one start event with an event definition.
- Boundary events have valid `attachedToRef`.

### 3. Validate each DMN file — apply all checks from `/audit-dmn`

For every `*.dmn` file:

#### 3a. Decision structure
- Each `<decision>` has unique non-empty `id` and `name`.
- No empty decision body.

#### 3b. Hit policy
- Valid hit policy value.
- `COLLECT` with aggregation: output column is numeric type.
- `PRIORITY`/`OUTPUT ORDER`: output column has `outputValues`.

#### 3c. Input columns
- `<inputExpression>` has non-empty `<text>`.
- `typeRef` is set.
- Rule input entries are subsets of `inputValues` if defined.

#### 3d. Output columns
- `<output name>` is non-empty.
- No duplicate output `name` values.
- `typeRef` is set.

#### 3e. Rules
- Each rule has correct count of `<inputEntry>` and `<outputEntry>` elements.
- No empty `<outputEntry>` in non-COLLECT tables (flags null output).
- No duplicate rules.

#### 3f. DRD references
- All `informationRequirement` hrefs resolve to existing decisions.
- No circular decision dependencies.

#### 3g. Cross-reference with BPMN
- `businessRuleTask` `zeebe:calledDecision decisionId` resolves to a DMN decision.
- `resultVariable` is non-empty.

### 4. Validate all FEEL expressions — apply all checks from `/check-feel`

Extract and validate every FEEL expression from all BPMN and DMN files:

- String literals use double quotes.
- Comparison uses `=` not `==`; logical uses `and`/`or` not `&&`/`||`.
- `conditionExpression` in BPMN starts with `=`.
- Range syntax uses `..` inside `[` or `(`.
- Date/time wrapped in FEEL function calls.
- No use of `%`, `^`, `!` operators.
- Unary tests (DMN input entries) are properly formed.
- No JavaScript-style operators or constructs.

### 5. Zeebe wiring validation — apply all checks from `/debug-zeebe`

- Job types are non-empty.
- Cross-reference job types against worker code if present.
- Message correlation keys are non-empty and not hardcoded.
- Error codes match between throw and catch.
- Call activity process IDs resolve locally where possible.
- Business rule task decision IDs resolve to DMN decisions.
- User tasks have assignment or candidature defined.
- `propagateAllChildVariables` does not contradict explicit output mappings.

### 6. Camunda 8 / Zeebe compatibility checks

Check for elements that are valid in BPMN 2.0 but **not supported by Zeebe**:

| Unsupported element | Common mistake |
|---|---|
| `businessRuleTask` without `zeebe:calledDecision` | Must reference a DMN decision |
| `scriptTask` without `zeebe:script` | Script tasks need `zeebe:script` extension |
| `sendTask` without `zeebe:taskDefinition` | Send tasks are treated as service tasks |
| `manualTask` | Not supported; use `userTask` |
| `adHocSubProcess` | Not supported in Zeebe |
| `complexGateway` | Not supported in Zeebe |
| Data objects (`dataObject`, `dataObjectReference`) | Ignored by Zeebe; flag if present |
| `dataStoreReference` | Not supported |
| Compensation events (`compensateEventDefinition`) | Not supported in Zeebe |
| Multi-instance with complex loop condition | Only sequential/parallel MI supported |
| `choreography` elements | Not supported |
| `collaboration` with multiple pools executing independently | Only one executable pool per deployment |

### 7. Process ID and version tag checks

- All process `id` attributes are non-empty and valid XML identifiers.
- No two BPMN files in the same deployment set define the same process `id` — this causes a deployment conflict.
- If `versionTag` is set on any process, note it; if not set, note that version management relies purely on deployment order.
- Decision `id` values are unique across all DMN files in the deployment set.

### 8. Best-practice warnings (non-blocking)

Flag these as informational warnings that don't block deployment but should be reviewed:

- Process has more than 100 elements — consider splitting into sub-processes or call activities.
- No error boundary events on any service task — unhandled job errors will create incidents with no automatic fallback.
- No retry configuration (`retries`) on any service task — relying on Zeebe default of 3.
- Call activity uses a hardcoded `processId` string (not a FEEL expression) — process version updates require BPMN changes.
- No message boundary event on any long-running service task — no way to cancel or redirect if the job hangs.
- Timer intermediate events with a hardcoded date (`timeDate`) that is in the past.
- User tasks with no time-out boundary — tasks can stay open indefinitely.
- Exclusive gateway immediately followed by another exclusive gateway with no logic between them — consider merging.

## Output format

Produce a structured, scannable report:

```
## 🚀 Camunda 8 Pre-Deployment Check

**Project:** <root directory>  
**Files scanned:** <N> BPMN, <M> DMN  
**Timestamp:** <current date/time>

---

## BPMN Files

### ✅ / ❌ <filename> (process: <id>)
- Errors: N  Warnings: M
- Key issues: <one-line summary of the worst finding, or "None">

[Repeat for each BPMN file]

---

## DMN Files

### ✅ / ❌ <filename>
- Errors: N  Warnings: M
- Key issues: <one-line summary>

[Repeat for each DMN file]

---

## FEEL Expression Summary
- Total expressions validated: N
- Errors: A  Warnings: B

---

## Zeebe Wiring Summary
- Job types defined: N  |  Workers found: M
- Unmatched job types: <list or "None">

---

## Compatibility Issues
- <list any unsupported BPMN elements found, or "None">

---

## ❌ All Errors (must fix before deployment)

| Severity | File | Element | Issue |
|---|---|---|---|
| ❌ ERROR | order.bpmn | Task_1 | Missing zeebe:taskDefinition type |
| ❌ ERROR | rules.dmn | decision[approval] rule[3] | outputEntry is empty |

---

## ⚠️ All Warnings (review before deployment)

| File | Element | Warning |
|---|---|---|
| order.bpmn | ExclusiveGateway_2 | No default flow — unmatched condition causes incident |

---

## 📋 Final Verdict

| Category | Errors | Warnings |
|---|---|---|
| BPMN Structure | N | M |
| DMN Logic | N | M |
| FEEL Expressions | N | M |
| Zeebe Wiring | N | M |
| Compatibility | N | M |
| **Total** | **N** | **M** |

### 🔴 NOT READY — Fix N error(s) before deploying.
### 🟡 REVIEW RECOMMENDED — No errors, but M warning(s) need attention.
### 🟢 READY TO DEPLOY — No errors or warnings found.
```

Use 🔴 if there are any errors, 🟡 if there are only warnings, and 🟢 if everything is clean.
