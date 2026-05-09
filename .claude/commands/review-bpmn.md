---
description: Review a Camunda 8 BPMN file for runtime bugs before deployment
allowed-tools: Read, Bash, Glob, Grep
argument-hint: <path/to/process.bpmn>
---

Review the Camunda 8 / Zeebe BPMN process file at `$ARGUMENTS` for runtime bugs that would cause failures after deployment. If no path is given, discover all `*.bpmn` files in the project with `find . -name "*.bpmn"` and review each one.

## Steps

1. **Read the file** — parse the raw XML and note the process ID, name, and element counts.

2. **Check structural integrity**
   - Exactly one None Start Event per top-level process (message/signal/timer starts are valid for event sub-processes and event-based gateways).
   - At least one End Event exists per pool/process.
   - Every SequenceFlow has both a `sourceRef` and a `targetRef` pointing to elements that exist in the model.
   - No element is completely unreachable (no incoming sequence flow and not a start event).
   - No element is a dead end (no outgoing sequence flow and not an end event or intermediate catch event waiting for an external trigger).

3. **Check gateway logic**
   - **Exclusive Gateway (XOR):** every outgoing flow (except the default flow) must have a `conditionExpression`. If there is no default flow, all branches must be exhaustive or a dead-path risk exists — flag this.
   - **Inclusive Gateway (OR):** same as XOR; also verify the gateway has more than one outgoing flow, otherwise it is pointless.
   - **Parallel Gateway (AND):** outgoing flows must NOT have conditions — Zeebe ignores them silently.
   - **Event-Based Gateway:** all outgoing flows must target intermediate catch events (message, signal, or timer), not tasks.

4. **Check Zeebe service task configuration**
   - Every `serviceTask` must have a `zeebe:taskDefinition` extension element with a non-empty `type` attribute.
   - `retries` should be a positive integer; flag if missing (defaults to 3, but explicit is safer).
   - Custom headers (`zeebe:taskHeaders` / `zeebe:header`) must not have duplicate `key` values.

5. **Check message elements**
   - Every `messageEventDefinition` must reference a `<message>` element with a non-empty `name` attribute.
   - Intermediate message catch events and receive tasks need a `zeebe:subscription` with a non-empty `correlationKey`.
   - Message start events inside an event sub-process also need a `zeebe:subscription correlationKey`.

6. **Check timer elements**
   - Timer start events need a valid `timeCycle` or `timeDate` expression.
   - Timer catch events need a valid `timeDuration` or `timeCycle` or `timeDate`.
   - Expressions must follow ISO 8601 or FEEL format (e.g. `PT1H`, `R3/PT10M`, `=date and time("2025-01-01T00:00:00")`); flag bare strings that look wrong.

7. **Check error elements**
   - Every `errorEventDefinition` on a boundary event must reference an `<error>` element with a non-empty `errorCode`.
   - End events with an error definition also need a valid `errorCode`.
   - Catch-all error boundary events (no `errorCode`) are valid but should be flagged as broad.

8. **Check signal elements**
   - Every `signalEventDefinition` must reference a `<signal>` element with a non-empty `name`.

9. **Check call activities**
   - Every `callActivity` must have a `zeebe:calledElement` extension element with a non-empty `processId`.
   - Flag if `propagateAllChildVariables` is `true` without comment — this can cause variable pollution.

10. **Check user tasks**
    - Every `userTask` should have either a `zeebe:formDefinition` (Camunda Forms) or a `formKey` (embedded form).
    - Flag user tasks with neither — they will create incidents in Tasklist.
    - Assignee and candidate group expressions should be non-empty if set.

11. **Check input/output variable mappings**
    - `zeebe:input` and `zeebe:output` elements must have both `source` and `target` attributes.
    - `source` values should be valid FEEL expressions (at minimum non-empty).
    - Duplicate `target` names within the same mapping block are likely bugs.

12. **Check sub-processes**
    - Embedded sub-processes must have their own start and end events.
    - Event sub-processes must have exactly one start event with an event definition (message, error, signal, or timer).
    - Boundary events must be attached (`attachedToRef`) to an existing element ID.

## Output format

Produce a structured report:

```
## BPMN Review: <process-id> (<filename>)

### ✅ Passed checks
- <list each passing check>

### ⚠️ Warnings  (valid but risky)
- [GATEWAY] <element-id>: Exclusive gateway has no default flow — unmatched conditions will cause an incident.
- ...

### ❌ Errors  (will cause runtime failure or incident)
- [SERVICE-TASK] <element-id> "<element-name>": Missing zeebe:taskDefinition type.
- [MESSAGE] <element-id>: messageEventDefinition references unknown message ID "<id>".
- ...

### 📋 Summary
- Errors: N   Warnings: M
- Verdict: READY TO DEPLOY / NEEDS FIXES BEFORE DEPLOY
```

For each finding include: severity emoji, category tag in `[BRACKETS]`, element ID, human-readable name, and a one-sentence explanation of the runtime impact.
