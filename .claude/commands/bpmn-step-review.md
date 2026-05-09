# BPMN Step Review

Systematically reviews every element in a BPMN process file against a production-readiness checklist. Run this before any deployment or after significant process changes.

## Usage

```
/bpmn-step-review [file-or-directory]
```

If no path is given, scan all `*.bpmn` files in the current directory tree.

## What This Checks

### Service Tasks
For every `<serviceTask>` and `<businessRuleTask>`:
- [ ] `retries` is set (default Zeebe is 3 — confirm it is intentional, not defaulted by accident)
- [ ] `retryBackoff` is configured if retries > 0
- [ ] Every expected input variable has a `zeebe:input` mapping with a non-empty `source`
- [ ] Every output variable consumed downstream has a matching `zeebe:output` mapping
- [ ] `resultVariable` is set on business rule tasks and matches the output mapping source
- [ ] Job type string is non-empty and follows the project naming convention

### Gateways
For every `<exclusiveGateway>` and `<inclusiveGateway>`:
- [ ] Exactly one outgoing sequence flow has no condition (the default path) — or the `default` attribute is set on the gateway
- [ ] All condition expressions use variables that are in scope at that point
- [ ] No dead branches (flows that can never be reached given the conditions)

### Call Activities
For every `<callActivity>`:
- [ ] `calledElement` references a deployed process key (not a local name)
- [ ] Every variable the sub-process needs is explicitly mapped in via `zeebe:input`
- [ ] Every variable the parent needs back is explicitly mapped out via `zeebe:output`
- [ ] Variable scope is isolated — the sub-process does not accidentally inherit parent variables unless that is deliberate

### Intermediate / Boundary Events
- [ ] Error boundary events have `errorCode` set (not a catch-all) unless a catch-all is deliberately intended
- [ ] Timer events use ISO 8601 format (`PT10S`, `P1D`, etc.) or a valid FEEL expression
- [ ] Message events have a `correlationKey` expression that will resolve at runtime

### General
- [ ] No orphan sequence flows (flows with no target)
- [ ] Every end event is reachable from the start event
- [ ] Process `id` and `name` are set and match the deployed artifact name

## Instructions

1. Locate the target BPMN files using the path argument or by searching the working directory.
2. For each file, parse the XML and walk every element listed above.
3. Report findings grouped by element type, then by file. Use this format:

```
## [filename.bpmn]

### Service Tasks
- ❌ "Reserve Inventory" — missing zeebe:output for `reservationId`
- ⚠️  "Send Confirmation Email" — retries=0, confirm this is intentional
- ✅ "Validate Order" — OK

### Gateways
...
```

4. After all files, print a summary: total elements checked, pass count, warning count, failure count.
5. If failures exist, list the top remediation actions in priority order.
