# Message Event Audit

Audits message start events, intermediate message catch events, and message boundary events. Use this when a process waits for external messages, receives asynchronous callbacks, or correlates events from another system.

## Usage

```
/message-event-audit [file-or-directory]
```

If no path is given, scan all `*.bpmn` files in the current directory tree.

## What This Checks

### Message Definitions
- [ ] Every message event references a declared `<message>` element
- [ ] Message names are non-empty and match the producer contract exactly
- [ ] Message names are stable business/event names, not environment-specific values
- [ ] Duplicate message names are intentional and documented

### Correlation Keys
- [ ] Every message catch or boundary event has a correlation key expression
- [ ] Correlation key expressions reference variables that are in scope before the wait state is entered
- [ ] Correlation keys use stable identifiers, not mutable status or display fields
- [ ] Correlation key type matches what the producer sends
- [ ] Null or missing correlation key cases are handled before the process waits

### Message Start Events
- [ ] Message start events define a message name that external producers can use
- [ ] Any expected payload variables are documented or mapped into the first consuming task
- [ ] Multiple message start events in one process have distinct message names
- [ ] Tenant or environment-specific routing assumptions are explicit if applicable

### Intermediate Catch Events
- [ ] The process writes the correlation variable before reaching the catch event
- [ ] No upstream gateway path can reach the catch event without the correlation variable
- [ ] The wait state has an expected timeout or operational recovery path when the message never arrives
- [ ] Downstream tasks do not assume optional message payload fields are always present

### Boundary Message Events
- [ ] Interrupting vs non-interrupting behavior is intentional
- [ ] If non-interrupting, downstream branches do not race on the same variables
- [ ] If interrupting, compensation or cleanup is handled when the attached activity is canceled
- [ ] Boundary message event payload variables do not overwrite important parent variables accidentally

## Reporting Format

```
## [filename.bpmn]

### Message Events
- ❌ "Wait for Shipment Callback" — correlation key `= shipmentId` can be null on the exception-handling path
- ⚠️  "Customer Updated" — non-interrupting boundary event writes `status`, which is also written by the main branch
- ✅ "Order Placed" — message start event has a stable message name and expected payload

### Message Definitions
- ⚠️  `shipment-callback-dev` — message name appears environment-specific
```

## Instructions

1. Locate target BPMN files using the path argument or by searching the working directory.
2. Parse all `<message>`, message start, intermediate catch, and boundary event definitions.
3. Resolve message references from each event to the declared message.
4. Trace correlation key variables back to write points and verify they are available on every path reaching the wait state.
5. Report findings grouped by file, then by message event.
6. Summarise: total message events checked, pass count, warning count, failure count.
