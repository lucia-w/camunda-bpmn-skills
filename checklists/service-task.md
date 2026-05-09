# Service Task Checklist

Copy this into a PR comment or review doc for each service task being reviewed.

---

**Task name:** ___________________________
**Job type:** ___________________________
**File:** ___________________________

## Configuration
- [ ] `retries` is set explicitly (not relying on default)
- [ ] If retries > 0: `retryBackoff` header is set
- [ ] Job type string is non-empty and follows kebab-case naming convention
- [ ] Job type matches what the worker actually subscribes to

## Inputs (`zeebe:input`)
- [ ] Every variable the worker reads is mapped in
- [ ] No input sources reference variables that won't be in scope at this point in the process
- [ ] No dead inputs (mapped in but worker doesn't use them)

## Outputs (`zeebe:output`)
- [ ] Every variable the process needs downstream is mapped out
- [ ] Output variable names don't shadow parent-scope variables unintentionally
- [ ] If this is a `businessRuleTask`: `resultVariable` is set and non-empty

## Error Handling
- [ ] Identified which exceptions the worker can throw
- [ ] Retryable exceptions: covered by retry config
- [ ] Business errors: covered by error boundary event with specific errorCode
- [ ] Unexpected errors: acceptable to incident (or explicitly handled)
- [ ] No catch-all boundary event unless intentional and documented

## Notes
___________________________
