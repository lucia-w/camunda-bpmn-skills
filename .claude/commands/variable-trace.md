# Variable Trace

Given a variable name, traces every point in the process where it is written and every point where it is read — across BPMN, DMN, and call activity boundaries. Use this to diagnose missing mappings, unexpected overwrites, or scope isolation bugs.

## Usage

```
/variable-trace <variableName> [file-or-directory]
```

Example: `/variable-trace paymentId`

## What This Traces

### Write points (where the variable is set)
- `zeebe:output` mappings on service tasks — `target` matches the variable name
- `zeebe:output` mappings on call activities — output from sub-process mapped into parent
- Process start: listed in the process's input variables / start form fields
- Message correlation payloads
- Script task output (if applicable)
- DMN `resultVariable` or `zeebe:output` from a business rule task

### Read points (where the variable is consumed)
- `zeebe:input` mappings on any task — `source` expression references the variable
- Gateway condition expressions — `= variableName == "someValue"` etc.
- `zeebe:input` on call activities — passed into a sub-process
- DMN input expressions that reference the variable
- `correlationKey` expressions on intermediate catch events
- Timer expressions that reference the variable

### Cross-boundary tracking
- If the variable crosses into a call activity via `zeebe:input`, trace continues inside the sub-process BPMN file
- If it crosses out via `zeebe:output`, note that the variable name may change (target vs source)
- Flag any call activity that consumes the variable on the way in but does NOT map it back out (potential scope trap)

## Reporting Format

```
## Tracing variable: `paymentId`

### Write Points
| Location | File | Element | Expression |
|----------|------|---------|------------|
| zeebe:output | order-fulfillment.bpmn | "Record Payment" (serviceTask) | target=`paymentId`, source=`= response.id` |
| call activity output | order-fulfillment.bpmn | "Process Payment" (callActivity) | ❌ NOT MAPPED — paymentId set inside sub-process but no zeebe:output brings it back |

### Read Points
| Location | File | Element | Expression |
|----------|------|---------|------------|
| zeebe:input | order-fulfillment.bpmn | "Send Payment Receipt" (serviceTask) | source=`= paymentId` |
| gateway condition | order-fulfillment.bpmn | "Payment Confirmed?" (exclusiveGateway) | `= paymentId != null` |

### Scope Issues
- ⚠️  `paymentId` is read in "Send Payment Receipt" but the only write point is inside a call activity with no output mapping — this will be `null` at runtime
```

## Instructions

1. Parse all BPMN and DMN files in scope.
2. Build two lists: write points and read points for the given variable name.
3. For each write point inside a call activity, check whether the variable is mapped out to the parent scope.
4. For each read point, check whether a write point precedes it on every path through the process (not just the happy path).
5. Report in the table format above, then summarise any scope issues found.
6. If no issues found: confirm the variable flows cleanly from write to read across all paths.
