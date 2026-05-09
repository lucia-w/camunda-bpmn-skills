# Gateway Checklist

Copy this into a PR comment or review doc for each gateway being reviewed.

---

**Gateway name:** ___________________________
**Gateway type:** [ ] Exclusive  [ ] Parallel  [ ] Inclusive  [ ] Event-based
**File:** ___________________________

## Exclusive Gateway
- [ ] Exactly one outgoing flow has no condition (default) OR the `default` attribute is set on the gateway
- [ ] Conditions are exhaustive — no gap between them where all evaluate to false
- [ ] Conditions are mutually exclusive — no overlap where two could both be true simultaneously
- [ ] All condition expressions reference variables that are in scope at this gateway
- [ ] No dead branches (conditions that can never be true given the process logic)

## Parallel Gateway
- [ ] All outgoing branches from the split will eventually reach a joining parallel gateway
- [ ] The join gateway has exactly as many incoming flows as the split has outgoing flows
- [ ] No branch can get stuck (e.g., on a timer or receive event that might never fire)

## Inclusive Gateway
- [ ] Default flow is defined (in case no conditions evaluate to true)
- [ ] The joining inclusive gateway accounts for variable numbers of active branches

## Condition Expression Quality
- [ ] Conditions use `=` FEEL prefix: `= variableName == "value"` not `variableName == "value"`
- [ ] Null checks are included where the variable might not be set: `= variableName != null && variableName > 0`
- [ ] No string conditions that should be boolean: `= status == "true"` vs `= status == true`

## Notes
___________________________
