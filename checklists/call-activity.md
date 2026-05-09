# Call Activity Checklist

Copy this into a PR comment or review doc for each call activity being reviewed.

---

**Activity name:** ___________________________
**Called process ID:** ___________________________
**File:** ___________________________

## Reference
- [ ] `calledElement` process ID exists in the codebase (or is a known deployed key)
- [ ] Process ID casing is correct (Zeebe is case-sensitive)

## Inputs (`zeebe:input`)
- [ ] Every variable the sub-process needs at startup is explicitly mapped in
- [ ] Source expressions reference variables that are in scope in the parent at this point
- [ ] No implicit inheritance assumed (`propagateAllParentVariables=true` default not relied upon)

## Outputs (`zeebe:output`)
- [ ] Every variable the parent needs after the call activity is explicitly mapped out
- [ ] Variable names in `target` don't accidentally overwrite parent variables
- [ ] `propagateAllChildVariables` is disabled or constrained by explicit output mappings unless broad propagation is intentional and documented

## Scope
- [ ] Variables that should stay inside the sub-process are not being mapped out
- [ ] The sub-process does not need any parent variables it isn't receiving via input mappings
- [ ] If the sub-process sets variables with the same name as parent variables, the intent is clear

## Error Handling
- [ ] If sub-process can throw BPMN errors, parent has matching error boundary events
- [ ] Error codes match between sub-process throw and parent catch

## Notes
___________________________
