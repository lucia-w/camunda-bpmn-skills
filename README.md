# Camunda BPMN Skills

Claude Code slash commands for reviewing, auditing, and debugging Camunda 8 / Zeebe BPMN and DMN processes.

Drop this repo anywhere Claude Code can see it — the skills in `.claude/commands/` become available as `/skill-name` commands.

---

## Install

Use this repository as a Claude Code project-level command pack:

1. Copy or clone it into the root of the BPMN/DMN project you want to review, or copy the `.claude/commands/` files into that project's `.claude/commands/` directory.
2. Restart Claude Code or reload the project so the slash commands are discovered.
3. Run commands from the target project root so they can scan local BPMN, DMN, form, and worker files.

---

## Skills

| Command | What it does |
|---------|-------------|
| [`/bpmn-step-review`](.claude/commands/bpmn-step-review.md) | Systematic checklist across all elements: retries, input/output mappings, gateway defaults, scope |
| [`/dmn-consistency-check`](.claude/commands/dmn-consistency-check.md) | Verifies every DMN input/output column is correctly wired to its business rule task |
| [`/variable-trace`](.claude/commands/variable-trace.md) | Given a variable name, traces every write and read across BPMN, DMN, and call activity boundaries |
| [`/call-activity-audit`](.claude/commands/call-activity-audit.md) | Checks sub-process references, input/output mappings, and scope isolation for all call activities |
| [`/error-handling-review`](.claude/commands/error-handling-review.md) | Audits error boundary events, error codes, catch-alls, and unhandled failure paths |
| [`/message-event-audit`](.claude/commands/message-event-audit.md) | Checks message names, correlation keys, payload assumptions, and boundary message behavior |
| [`/timer-event-audit`](.claude/commands/timer-event-audit.md) | Checks timer formats, timer expressions, timeout paths, and interrupting boundary behavior |
| [`/multi-instance-review`](.claude/commands/multi-instance-review.md) | Reviews multi-instance collection wiring, item scope, output aggregation, and parallel write conflicts |
| [`/feel-expression-review`](.claude/commands/feel-expression-review.md) | Reviews FEEL expressions for syntax, missing variables, type mismatches, and null safety |
| [`/worker-contract-audit`](.claude/commands/worker-contract-audit.md) | Cross-checks service task job types, inputs, outputs, headers, retries, and worker error codes |
| [`/form-user-task-review`](.claude/commands/form-user-task-review.md) | Reviews user task assignment, form references, form variables, and human workflow paths |
| [`/process-versioning-audit`](.claude/commands/process-versioning-audit.md) | Checks process, decision, form, and call activity references for deployment-time versioning risks |
| [`/deployment-preflight`](.claude/commands/deployment-preflight.md) | Runs all checks in sequence and produces a go/no-go deploy report |

---

## Quick Start

```bash
# Review all BPMN files in the current directory
/bpmn-step-review

# Check a specific process
/bpmn-step-review src/main/resources/loan-approval.bpmn

# Check DMN wiring
/dmn-consistency-check src/main/resources/

# Trace where a variable comes from and goes to
/variable-trace vaultId

# Check message event correlation
/message-event-audit src/main/resources/

# Review FEEL expressions across BPMN and DMN
/feel-expression-review src/main/resources/

# Full pre-deploy check
/deployment-preflight src/main/resources/
```

---

## Supporting Docs

- [Zeebe Variable Scope](docs/zeebe-variable-scope.md) — how variable scope works across processes and call activities
- [DMN Integration Guide](docs/dmn-integration-guide.md) — best practices for wiring DMN to BPMN
- [Error Handling Patterns](docs/error-handling-patterns.md) — when to use boundary events vs retries vs incidents
- [BPMN Common Anti-patterns](docs/bpmn-antipatterns.md) — patterns that pass validation but break at runtime

## Checklists

Printable/copy-pasteable checklists for PR reviews and pair reviews:

- [Service Task Checklist](checklists/service-task.md)
- [Call Activity Checklist](checklists/call-activity.md)
- [Gateway Checklist](checklists/gateway.md)
- [DMN Integration Checklist](checklists/dmn-integration.md)


