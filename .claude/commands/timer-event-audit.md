# Timer Event Audit

Audits timer start events, intermediate timer catch events, and timer boundary events. Use this to catch invalid timer expressions, accidental cancellation behavior, and missing timeout recovery paths.

## Usage

```
/timer-event-audit [file-or-directory]
```

If no path is given, scan all `*.bpmn` files in the current directory tree.

## What This Checks

### Timer Expression Format
- [ ] Timer definitions use exactly one of `timeDuration`, `timeDate`, or `timeCycle`
- [ ] Static durations use ISO 8601 format, e.g. `PT10S`, `PT5M`, `P1D`
- [ ] Date values are valid ISO 8601 timestamps or valid FEEL expressions
- [ ] Cycle values are intentional and documented for repeated timers
- [ ] FEEL timer expressions start with `=` and reference variables in scope

### Boundary Timers
- [ ] Interrupting vs non-interrupting behavior is intentional
- [ ] Interrupting timers have cleanup or compensation when the attached activity is canceled
- [ ] Non-interrupting timers do not create unbounded repeated side effects
- [ ] Timeout branches lead to a meaningful process outcome, retry, escalation, or incident path

### Intermediate Timers
- [ ] Timer waits are reachable only when waiting is actually required
- [ ] Downstream variables do not assume work happened while the token was waiting
- [ ] Timer duration variables are written before the token reaches the timer
- [ ] Long waits have an operational explanation or runbook expectation

### Timer Start Events
- [ ] Scheduled process starts have a clear owner and cadence
- [ ] `timeCycle` schedules cannot accidentally start duplicate overlapping instances
- [ ] Time zone assumptions are explicit when schedules depend on local business time

## Reporting Format

```
## [filename.bpmn]

### Timer Events
- ❌ "Wait 10 seconds" — uses `10 seconds`; expected ISO 8601 duration such as `PT10S`
- ⚠️  "Order Timeout" — interrupting boundary timer cancels the task with no cleanup path
- ✅ "Daily Batch Start" — cycle expression is valid and documented
```

## Instructions

1. Locate target BPMN files using the path argument or by searching the working directory.
2. Parse all timer start, intermediate catch, and boundary event definitions.
3. Validate timer expression type and format.
4. Trace any FEEL variables used by timer expressions and verify they are in scope.
5. Review interrupting/non-interrupting timer behavior and downstream effects.
6. Summarise: total timer events checked, pass count, warning count, failure count.
