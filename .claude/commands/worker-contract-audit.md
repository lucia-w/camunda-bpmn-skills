# Worker Contract Audit

Audits the contract between BPMN service tasks and worker implementations. Use this when worker code is available in the same repository or when the expected worker behavior is documented.

## Usage

```
/worker-contract-audit [file-or-directory]
```

If no path is given, scan all BPMN files and nearby source files in the current directory tree.

## What This Checks

### Job Type Contract
- [ ] Every service task has a non-empty job type
- [ ] Job type casing and separators match the worker subscription exactly
- [ ] No unused worker subscriptions exist for removed BPMN tasks
- [ ] No two unrelated task contracts share the same job type accidentally

### Input Contract
- [ ] Every variable the worker reads is supplied by `zeebe:input` or is known to be in scope
- [ ] Input variable names match worker code exactly
- [ ] Required inputs are validated before side effects happen
- [ ] Sensitive inputs are not passed where they are not needed

### Output Contract
- [ ] Every variable the worker completes with is captured by `zeebe:output` when downstream process logic needs it
- [ ] Output variable names match downstream task and gateway usage
- [ ] Worker output shape matches FEEL property access in BPMN
- [ ] Optional outputs are handled safely downstream

### Errors and Retries
- [ ] Worker business errors match BPMN `<error>` codes exactly
- [ ] Retryable exceptions map to retry configuration and backoff
- [ ] Non-retryable failures either throw BPMN errors or intentionally create incidents
- [ ] Worker timeout, idempotency, and duplicate job handling are understood

### Headers and Configuration
- [ ] Required custom headers are present and non-empty
- [ ] Header values match what the worker expects
- [ ] Retry/backoff headers are not confused with worker-specific headers

## Reporting Format

```
## [filename.bpmn]

### Worker Contracts
- ❌ "Reserve Inventory" — BPMN job type `inventory-reserve`; worker subscribes to `inventoryReserve`
- ❌ "Record Payment" — worker returns `payment.id`, but BPMN output reads `= response.id`
- ⚠️  "Send Confirmation Email" — worker can throw `INVALID_EMAIL`, but no boundary event catches it
- ✅ "Validate Order" — inputs, outputs, headers, and error codes match worker contract
```

## Instructions

1. Locate target BPMN files and inspect service task job types, mappings, headers, retries, and boundary errors.
2. Search source files for worker subscriptions, job handlers, completion variables, and BPMN error throws.
3. Cross-reference job types, input names, output names, custom headers, and error codes.
4. If worker code is not available, report the BPMN-side contract and list assumptions that need confirmation.
5. Report findings grouped by BPMN file and service task.
6. Summarise: total service tasks checked, pass count, warning count, failure count.
