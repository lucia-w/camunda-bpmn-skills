# camunda-bpmn-skills

Claude Code skill commands for reviewing, auditing, and debugging **Camunda 8 / Zeebe** BPMN and DMN processes — catch runtime bugs before deployment.

## Commands

| Command | Description |
|---|---|
| `/review-bpmn` | Scan a BPMN file for structural and configuration bugs (gateways, events, task setup) |
| `/audit-dmn` | Audit a DMN decision table for correctness, coverage gaps, and FEEL expression issues |
| `/debug-zeebe` | Debug Zeebe-specific wiring (job types, variable mappings, message correlation, error handling) |
| `/check-feel` | Extract and validate every FEEL expression across BPMN and DMN files |
| `/pre-deploy-check` | Run all checks in one pass and produce a go/no-go deployment verdict |

## Installation

Copy the `.claude/` directory into your Camunda project root:

```bash
# From your Camunda project directory:
git clone https://github.com/lucia-w/camunda-bpmn-skills /tmp/camunda-bpmn-skills
cp -r /tmp/camunda-bpmn-skills/.claude .
```

Or, if you use the [Claude Code](https://docs.anthropic.com/en/docs/claude-code) shared commands directory (`~/.claude/commands/`), copy the files there to make them available in every project.

## Usage

### `/review-bpmn` — Review a BPMN file

```
/review-bpmn src/main/resources/order-process.bpmn
```

Checks for:
- Missing or extra Start/End Events
- Broken SequenceFlow references (dangling arrows)
- Exclusive/Inclusive Gateway conditions and default flows
- Parallel Gateway conditions (conditions are silently ignored by Zeebe)
- `zeebe:taskDefinition type` missing on Service Tasks
- Message names and correlation keys
- Timer expression format (ISO 8601 or FEEL)
- Error codes on Error Boundary Events
- Signal names on Signal Events
- `zeebe:calledElement processId` on Call Activities
- Form binding on User Tasks
- Input/output mapping completeness
- Sub-process and Event Sub-process structure

### `/audit-dmn` — Audit a DMN file

```
/audit-dmn src/main/resources/loan-approval.dmn
```

Checks for:
- Hit policy correctness and aggregation type compatibility
- Input column `inputExpression` and `typeRef`
- Output column `name`, `typeRef`, and `outputValues` (required for PRIORITY/OUTPUT ORDER)
- Rule count mismatches (wrong number of input/output entries)
- Empty output entries producing silent `null` returns
- FEEL syntax in input and output entries
- Duplicate rules
- Coverage gaps for exhaustive tables
- Circular decision dependencies in the DRD
- Cross-reference with BPMN Business Rule Task `decisionId` references

### `/debug-zeebe` — Debug Zeebe wiring

```
/debug-zeebe src/main/resources/
```

Checks for:
- Job types that are empty or differ only by case
- Job types in BPMN not matched by any worker in the codebase
- Workers in the codebase not matched by any job type in BPMN
- Input mapping `source` and `target` validity (FEEL paths, no dot-notation in target)
- Output mapping source/target validity and duplicate targets
- Hardcoded message correlation keys
- Message start event correlation keys (should be absent)
- Error code mismatches between `ZeebeBpmnError` throws and boundary event catches
- Catch-all error boundaries that swallow specific errors
- `propagateAllChildVariables` used alongside explicit `zeebe:output` mappings
- Call activity process IDs that can't be resolved locally
- Business Rule Task `decisionId` references that can't be resolved
- User tasks with no assignee, candidate groups, or candidate users

### `/check-feel` — Validate FEEL expressions

```
/check-feel src/main/resources/
```

Validates every FEEL expression found in BPMN and DMN files:
- Unquoted strings (`pending` → `"pending"`)
- Wrong equality operator (`==` → `=`)
- Wrong logical operators (`&&`/`||` → `and`/`or`)
- Wrong negation (`!` → `not(...)`)
- `conditionExpression` without `=` prefix (treated as string literal, always `true`)
- Unary test format in DMN input entries
- Date literals not wrapped in `date()`/`date and time()` functions
- Invalid range syntax (`[1-10]` instead of `[1..10]`)
- Modulo using `%` instead of `modulo()` function
- JavaScript-style ternary `? :` instead of `if/then/else`
- Bare ISO 8601 date strings that evaluate as arithmetic

### `/pre-deploy-check` — Full pre-deployment check

```
/pre-deploy-check
# or for a specific directory:
/pre-deploy-check src/main/resources/
```

Runs all of the above checks plus:
- Zeebe compatibility (flags unsupported BPMN 2.0 elements: `manualTask`, `adHocSubProcess`, `complexGateway`, compensation events, etc.)
- Unique process IDs across all BPMN files (duplicate IDs cause deployment conflicts)
- Unique decision IDs across all DMN files
- Version tag presence
- Best-practice warnings: no error boundaries on service tasks, missing timeouts on user tasks, timers with past dates, etc.

Produces a final **🔴 NOT READY / 🟡 REVIEW RECOMMENDED / 🟢 READY TO DEPLOY** verdict.

## What these skills catch

| Category | Example bugs caught |
|---|---|
| **Broken flow** | SequenceFlow with missing target; unreachable task |
| **Gateway** | XOR gateway with no default flow and non-exhaustive conditions |
| **Service task** | Empty `zeebe:taskDefinition type`; job never activated |
| **Messages** | Empty correlation key; message start event with correlation key |
| **Timers** | `timeDuration` set to `"1 day"` instead of `"P1D"` |
| **Errors** | Boundary event with no `errorCode`; mismatch with worker throw |
| **DMN coverage** | Boolean input column with only `true` rules; `false` case returns `null` |
| **FEEL syntax** | `amount == 100` (uses `==`); `date > "2025-01-01"` (bare date string) |
| **Zeebe wiring** | Worker registered for `payment-svc` but BPMN has `payment-service` |
| **Compatibility** | `manualTask` used (not supported by Zeebe) |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) with access to `Read`, `Bash`, `Glob`, and `Grep` tools.
- Camunda 8 / Zeebe BPMN 2.0 files (XML).
- DMN 1.3 files (XML).
