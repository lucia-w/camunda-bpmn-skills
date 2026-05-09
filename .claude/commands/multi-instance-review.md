# Multi-Instance Review

Reviews multi-instance activities and sub-processes for collection wiring, item-level variable scope, output aggregation, and parallel branch conflicts.

## Usage

```
/multi-instance-review [file-or-directory]
```

If no path is given, scan all `*.bpmn` files in the current directory tree.

## What This Checks

### Collection Wiring
- [ ] `inputCollection` is set and references a variable in scope
- [ ] `inputElement` is set and has a clear item-level name
- [ ] The activity's `zeebe:input` mappings use the item-level variable where appropriate
- [ ] Empty collection behavior is intentional and downstream-safe
- [ ] Collection variable type is list-like at runtime

### Sequential vs Parallel
- [ ] Parallel multi-instance activities do not write to the same top-level variable from each instance
- [ ] Sequential multi-instance activities do not rely on accidental variable carry-over from previous iterations
- [ ] Worker side effects are idempotent when instances can run in parallel
- [ ] Completion conditions are valid FEEL expressions and safe when zero items complete

### Output Aggregation
- [ ] `outputCollection` is set when downstream logic needs per-item results
- [ ] `outputElement` references a value produced by each instance
- [ ] Aggregated result names do not shadow the input collection name
- [ ] Downstream tasks treat aggregated outputs as lists, not scalars

### Scope and Errors
- [ ] Variables written inside one instance are not read by another instance unless explicitly aggregated
- [ ] Boundary error events on multi-instance activities have intentional cancel behavior
- [ ] Incidents in one instance have an understood effect on the overall activity

## Reporting Format

```
## [filename.bpmn]

### Multi-Instance Activities
- ❌ "Check Each Item" — parallel instances all write `availabilityStatus`; use item-level output aggregation instead
- ⚠️  "Notify Recipients" — no `outputCollection`; confirm results are intentionally discarded
- ✅ "Validate Documents" — input collection, item variable, and output aggregation are consistent
```

## Instructions

1. Locate target BPMN files using the path argument or by searching the working directory.
2. Find all elements with `multiInstanceLoopCharacteristics`.
3. Inspect Zeebe multi-instance extension attributes and input/output mappings.
4. Trace variables read and written inside the multi-instance body.
5. Flag shared writes, missing aggregation, unsafe completion conditions, and unclear empty-collection behavior.
6. Summarise: total multi-instance activities checked, pass count, warning count, failure count.
