# Form and User Task Review

Reviews user tasks and forms for assignment, form references, variable contracts, and downstream process assumptions.

## Usage

```
/form-user-task-review [file-or-directory]
```

If no path is given, scan all `*.bpmn` files and form files in the current directory tree.

## What This Checks

### User Task Configuration
- [ ] Every user task has a clear name and purpose
- [ ] Assignment, candidate users, candidate groups, or expressions are set intentionally
- [ ] Assignment expressions reference variables in scope
- [ ] Due date and follow-up date expressions are valid when used
- [ ] Priority expressions are valid and bounded when used

### Form References
- [ ] Referenced forms exist in the repository or are known deployed forms
- [ ] Form IDs, keys, or references match exactly
- [ ] Embedded forms and linked forms are used consistently with the project convention
- [ ] Form schema versions are compatible with the deployed process version

### Form Variables
- [ ] Every form field output used downstream is named consistently
- [ ] Required form fields are validated before downstream automated tasks rely on them
- [ ] Optional form fields are handled with null-safe FEEL expressions
- [ ] Form field names do not shadow existing process variables accidentally
- [ ] Sensitive form fields are not propagated beyond their required scope

### Human Workflow
- [ ] Rework, reject, timeout, or escalation paths are modeled where needed
- [ ] User task completion cannot skip required downstream data
- [ ] Boundary timers or messages on user tasks have intentional interrupting behavior
- [ ] Manual decisions are reflected in explicit variables used by gateways

## Reporting Format

```
## [filename.bpmn]

### User Tasks
- ❌ "Manual Approval" — gateway reads `approvalDecision`, but no form field or output mapping writes it
- ⚠️  "Review Order" — candidate group expression references `region`, which may be null
- ✅ "Confirm Delivery Details" — form reference exists and downstream variables are validated
```

## Instructions

1. Locate BPMN and form files using the path argument or by searching the working directory.
2. Find all user tasks and inspect assignment, form references, due dates, and custom headers.
3. Resolve form references where form files are available.
4. Trace form output variables into downstream tasks, gateways, and DMN decisions.
5. Report missing forms, assignment issues, unsafe optional fields, and missing human recovery paths.
6. Summarise: total user tasks checked, pass count, warning count, failure count.
