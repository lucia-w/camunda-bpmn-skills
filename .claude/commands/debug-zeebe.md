---
description: Debug Zeebe-specific wiring issues in a Camunda 8 process (job types, mappings, messages, errors)
allowed-tools: Read, Bash, Glob, Grep
argument-hint: <path/to/process.bpmn or directory>
---

Debug Zeebe-specific wiring issues in the Camunda 8 BPMN process(es) at `$ARGUMENTS`. If a directory is given, scan all `*.bpmn` files inside it. If no argument is given, scan the current project.

Focus exclusively on Zeebe runtime-wiring concerns that are not caught by schema validation alone — things that compile fine but fail at runtime.

## Steps

### 1. Inventory all Zeebe extension elements

Scan every BPMN file and build an inventory:

```bash
# Example greps (adapt to actual file paths)
grep -rn "zeebe:taskDefinition\|zeebe:calledElement\|zeebe:calledDecision\|zeebe:subscription\|zeebe:input\|zeebe:output\|zeebe:header\|zeebe:formDefinition" "$ARGUMENTS"
```

Collect:
- **Job types** — all distinct `zeebe:taskDefinition type` values and which elements use them.
- **Called processes** — all `zeebe:calledElement processId` values.
- **Called decisions** — all `zeebe:calledDecision decisionId` and `resultVariable`.
- **Message names and correlation keys** — all `<message name>` and `zeebe:subscription correlationKey` pairs.
- **Error codes** — all `<error errorCode>` values referenced by boundary events and end events.
- **Signal names** — all `<signal name>` values.
- **Input mappings** — all `zeebe:input source → target` pairs per element.
- **Output mappings** — all `zeebe:output source → target` pairs per element.
- **Headers** — all `zeebe:header key/value` pairs per task.

### 2. Job type consistency checks

- Flag any `zeebe:taskDefinition` with an **empty or whitespace-only `type`** — this will prevent the job from being picked up by any worker.
- If the project includes worker code (Java, Node.js, Go, etc.), search for job type strings registered in the workers:
  ```bash
  grep -rn "@ZeebeWorker\|JobWorkerValue\|zbc.NewJobWorker\|client.newWorker\|createWorker" .
  grep -rn "type\s*=\|jobType\|job_type\|jobKey" . --include="*.java" --include="*.go" --include="*.js" --include="*.ts"
  ```
  Cross-reference: flag any job type defined in BPMN but not handled by any worker, and vice versa.
- Flag job types that differ only in case (e.g., `processPayment` vs `ProcessPayment`) — Zeebe job types are case-sensitive.
- Flag duplicate job type definitions if the same type is used in two **different** process contexts where isolation is expected.

### 3. Input/output variable mapping checks

For every element with `zeebe:inputOutput`:

- **Input mapping source** must be a valid FEEL path or expression. Check:
  - Not empty.
  - Starts with a valid identifier or `=` (for expressions).
  - No unbalanced brackets or quotes.
  - Flag `=` prefix on source that contains a bare variable name — `=myVar` is valid FEEL but may be confusing; bare `myVar` (without `=`) is also valid as a FEEL path.
- **Input mapping target** must be a simple variable name (no dots, no FEEL syntax). Flag if it contains dots or special characters — Zeebe will create the variable with the dot literally, not as a nested path.
- **Output mapping source** must be a valid FEEL path relative to the job's output variables.
  - Flag `=` expressions that reference variables that don't appear to be produced by the job (if worker code is available).
- **Output mapping target** same rules as input target.
- Flag **duplicate target names** within the same `zeebe:inputOutput` block — the last mapping wins silently.
- Flag **output mappings with `resultVariable`** set on a Business Rule Task when `zeebe:calledDecision` is also present and the output mapping overwrites the same variable — this may cause confusion.

### 4. Message correlation checks

For each message catch event (intermediate or boundary) and receive task:

- `<message name>` must be non-empty and match the name used by the throwing side (process API call or BPMN message throw event in another process).
- `zeebe:subscription correlationKey` must be a non-empty FEEL expression that evaluates to a value present in process instance variables at the time the event is expected.
  - Flag bare names that don't exist as obvious variable names in the preceding flow.
  - Flag hardcoded strings as correlation keys (e.g., `="fixed-value"`) — they usually indicate a copy-paste bug where all instances would correlate to the same subscriber.
- Start message events that create new instances do NOT need a correlation key — flag if they have one set (it's ignored but confusing).
- Flag message names that appear in BPMN as both a start event trigger and an intermediate catch event in the same deployment — this can cause unintended instance creation.

### 5. Error boundary event and error end event checks

- Every `<errorEventDefinition>` on a boundary event must reference an `<error>` element. Flag missing `errorRef`.
- Every referenced `<error>` must have a non-empty `errorCode`. Flag missing or empty `errorCode`.
- Check that error codes thrown by service tasks (look for `throw new ZeebeBpmnError(...)` or equivalent in worker code) match the error codes defined in boundary events of the corresponding tasks.
- Flag **catch-all** error boundary events (no `errorCode`) — they are valid but swallow all errors; flag as a warning when a more specific handler exists on the same element.
- An error boundary event that is **non-interrupting** (`cancelActivity="false"`) is valid for certain patterns, but flag it if the task it's attached to is a service task — non-interrupting error boundaries on service tasks are unusual and may be a misconfiguration.

### 6. Signal checks

- Every `<signalEventDefinition>` must reference a `<signal>` with a non-empty `name`.
- Signal broadcast (throw) and signal catch (catch event) names must match exactly — case-sensitive.
- Flag signals used in both a start event and an intermediate catch event in the same process — this may create ambiguous routing.

### 7. Timer expression checks

- `timeDuration` values: must be ISO 8601 duration (`PT10S`, `P1DT2H`) or a FEEL expression starting with `=`.
- `timeCycle` values: must be ISO 8601 repeating interval (`R3/PT1H`) or a cron expression, or a FEEL expression.
- `timeDate` values: must be ISO 8601 date-time (`2025-06-01T09:00:00Z`) or a FEEL date-time expression.
- Flag timer expressions that are plain integers or bare strings without a format prefix — they will cause a deployment or activation error.

### 8. Call activity process ID checks

- Every `zeebe:calledElement processId` must be a non-empty string.
- If it is a FEEL expression (starts with `=`), flag it as dynamic — dynamic called element IDs are only supported in Camunda 8.6+.
- Search the project for the referenced process ID in other BPMN files. Flag IDs that cannot be resolved locally (they may be deployed separately, but unresolvable references should be documented).
- `propagateAllChildVariables="true"` with `zeebe:output` mappings on the same call activity is contradictory — flag it.

### 9. Business Rule Task (DMN) checks

- Every `zeebe:calledDecision` must have:
  - Non-empty `decisionId`.
  - Non-empty `resultVariable`.
- Search DMN files in the project for the referenced `decisionId`. Flag missing decisions.
- If `resultVariable` clashes with an existing process variable name that was mapped as an input, flag the potential overwrite.

### 10. User task assignment checks

- `zeebe:assignmentDefinition` `assignee` and `candidateGroups` must be non-empty if the attribute is present.
- FEEL expressions in assignee/candidateGroups must be properly quoted strings or valid FEEL paths (starting with `=`).
- Flag user tasks with neither `assignee`, `candidateGroups`, nor `candidateUsers` — unassigned tasks will appear in Tasklist but nobody will be notified.

## Output format

```
## Zeebe Wiring Debug Report

### Job Types Inventory
| Job Type | Element ID | Element Name |
|---|---|---|
| payment-service | Task_1a2b | Charge Customer |

### Unhandled Job Types (in BPMN but no worker found)
- `legacy-email-sender` (Task_3x9z "Send Welcome Email")

### Unregistered Workers (in code but no BPMN task found)
- `audit-logger` (workers/AuditWorker.java:42)

### ❌ Errors
- [JOB-TYPE] Task_5f2a "Process Refund": empty zeebe:taskDefinition type — job will never be activated.
- [CORRELATION-KEY] IntermediateEvent_8g3b "Wait for Payment": correlationKey is empty.
- [ERROR-CODE] BoundaryEvent_2h4c: errorRef points to <error> with no errorCode.
- ...

### ⚠️ Warnings
- [CATCH-ALL-ERROR] BoundaryEvent_9j1d: no errorCode — catches all errors from "Call External API".
- [PROPAGATE-VARS] CallActivity_4k5m: propagateAllChildVariables=true with explicit zeebe:output mappings — output mappings will be ignored.
- [CORRELATION-HARDCODED] IntermediateEvent_7n6p: correlationKey="fixed-id" is a hardcoded string.
- ...

### 📋 Summary
- Errors: N  Warnings: M
- Verdict: WIRING OK / WIRING ISSUES FOUND
```
