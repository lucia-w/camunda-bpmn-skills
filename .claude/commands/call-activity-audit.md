# Call Activity Audit

Audits all call activities in a process — verifying that sub-process references exist, input/output mappings are complete, and variable scope is correctly isolated between parent and child processes.

## Usage

```
/call-activity-audit [file-or-directory]
```

## What This Checks

### Reference Integrity
- [ ] `calledElement` value matches a `process/@id` that exists in the codebase (or is a known deployed key)
- [ ] The referenced process file is reachable from the current directory tree
- [ ] No typos in process IDs (case-sensitive match)

### Input Mappings (`zeebe:input`)
- [ ] Every variable the sub-process reads at startup is explicitly mapped in — do not rely on implicit variable inheritance
- [ ] No `zeebe:input` source expression references a variable that is not in scope in the parent at the point the call activity is reached
- [ ] No redundant mappings (input mapped in but sub-process never reads it)

### Output Mappings (`zeebe:output`)
- [ ] Every variable the parent needs after the call activity completes is explicitly mapped out
- [ ] No variable set inside the sub-process silently leaks into the parent scope without a mapping
- [ ] After the call activity, every downstream task that reads a variable from the sub-process result has a corresponding `zeebe:output` mapping

### Scope Isolation
- [ ] Sub-process does not rely on implicit parent variable propagation — `propagateAllParentVariables` defaults to `true`, but explicit inputs are safer
- [ ] If `propagateAllParentVariables=true` is set or omitted, flag it when the child should receive only a narrow input contract
- [ ] If `propagateAllChildVariables=true` is set or omitted without explicit output mappings, flag it — child variables may leak into the parent scope
- [ ] Variables set in the sub-process stay inside the sub-process unless explicitly mapped out

### Error Propagation
- [ ] If the sub-process can throw a BPMN error, the parent call activity has a corresponding error boundary event
- [ ] Error codes thrown inside the sub-process match what the parent boundary event catches

## Reporting Format

```
## [parent-process.bpmn]

### Call Activity: "Process Payment" → sub-process: payment-processing.bpmn

Reference
- ✅ Process ID `payment-processing` found in payment-processing.bpmn

Inputs
- ✅ `orderId` mapped in
- ❌ `orderAmount` — sub-process reads this at "Calculate Payment Fee" but no zeebe:input maps it in

Outputs
- ❌ `paymentId` — set in sub-process "Record Payment" but no zeebe:output maps it back to parent
- ✅ `paymentStatus` — mapped out correctly

Scope
- ⚠️  `propagateAllChildVariables=true` is set or defaulted — all sub-process variables may leak into parent scope
```
