# DMN Integration Checklist

Copy this into a PR comment or review doc for each business rule task + DMN table pair being reviewed.

---

**Task name:** ___________________________
**Decision ID (`decisionRef`):** ___________________________
**DMN file:** ___________________________
**Result variable:** ___________________________

## Decision Reference
- [ ] `decisionRef` value matches `decision/@id` in the DMN file exactly (case-sensitive)
- [ ] DMN file is deployed alongside the BPMN process

## DMN Inputs → BPMN Inputs
For each DMN input column:

| DMN Input Expression | Matching `zeebe:input` target | In scope at task? |
|---------------------|-------------------------------|-------------------|
| | | |
| | | |
| | | |

- [ ] Every DMN input expression has a corresponding `zeebe:input` mapping
- [ ] Variable name casing matches exactly between `zeebe:input target` and DMN input expression
- [ ] No DMN input references a variable that won't be in scope when the task runs

## DMN Outputs → BPMN Outputs
For each DMN output column:

| DMN Output Column | Extracted via `zeebe:output`? | Used downstream? |
|------------------|-------------------------------|-----------------|
| | | |
| | | |

- [ ] `resultVariable` is set and non-empty
- [ ] Every DMN output column that is used downstream is extracted via `zeebe:output`
- [ ] Unused output columns are intentionally discarded (not accidentally missed)

## Hit Policy
- [ ] Hit policy: ___________________________
- [ ] If COLLECT or RULE ORDER: downstream usage handles a list, not a scalar
- [ ] If UNIQUE or FIRST: downstream usage treats result as a scalar

## Testing
- [ ] Decision table tested in isolation with representative inputs
- [ ] Wiring tested end-to-end in a process instance test

## Notes
___________________________
