# Error Handling Review

Audits all error, escalation, and compensation handling in a BPMN process. Checks that errors are caught at the right scope, error codes are specific, and no failure path ends without a handled outcome.

## Usage

```
/error-handling-review [file-or-directory]
```

## What This Checks

### Error Boundary Events
- [ ] Every `<errorEventDefinition>` on a boundary event has an `errorRef` pointing to a declared `<error>` element — not a catch-all (empty errorRef)
- [ ] Catch-all boundaries (no errorRef) are flagged as warnings — they swallow unexpected errors silently
- [ ] The error code in the `<error>` element matches what the worker actually throws (cross-reference with worker job type naming convention)
- [ ] After catching an error, the flow leads to a meaningful compensation path — not directly to an end event with no action

### Escalation Events
- [ ] `<escalationEventDefinition>` has an `escalationRef` pointing to a declared `<escalation>`
- [ ] Escalation is used for non-exceptional flows (e.g. approval escalation) — not as a workaround for error handling
- [ ] Escalation boundary events on sub-processes don't silently cancel the sub-process unless that is the intent

### Compensation
- [ ] Compensation boundary events are attached only to tasks that have a compensation handler activity
- [ ] Compensation handlers don't themselves throw uncaught errors
- [ ] `<compensateEventDefinition>` in the flow is triggered intentionally, not reachable by accident

### Unhandled Paths
- [ ] Every service task that can fail has either: a boundary error event, or a retry strategy that eventually succeeds, or documented acceptance that it will incident
- [ ] Incident-producing failures (exhausted retries) are acceptable for this task — not silently lost
- [ ] No terminal error end events without a corresponding boundary catch somewhere in the parent scope

### Retry vs Error Handling
- [ ] Tasks with retries > 0 should NOT also have a boundary error event for the same error code (double-handling is confusing)
- [ ] Tasks with retries = 0 that can fail should always have a boundary error event

## Reporting Format

```
## [filename.bpmn]

### Error Boundary Events
- ❌ "Reserve Inventory" — catch-all boundary (no errorRef); will swallow any worker exception
- ⚠️  "Validate Order" — errorCode `VALIDATION_FAILED` declared but worker throws `validation-failed` (case mismatch)
- ✅ "Charge Payment" — errorCode `PAYMENT_GATEWAY_UNAVAILABLE`, recovery flow leads to manual review

### Unhandled Failure Paths
- ❌ "Send Confirmation Email" — retries=0, no boundary event; failure will create an incident with no recovery path documented
```
