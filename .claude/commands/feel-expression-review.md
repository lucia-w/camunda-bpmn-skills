# FEEL Expression Review

Reviews FEEL expressions in BPMN and DMN files for syntax hazards, missing variables, type mismatches, and nullable comparisons.

## Usage

```
/feel-expression-review [file-or-directory]
```

If no path is given, scan all `*.bpmn` and `*.dmn` files in the current directory tree.

## What This Checks

### Expression Locations
- [ ] BPMN sequence flow conditions
- [ ] Zeebe input and output mappings
- [ ] Job headers that contain FEEL expressions
- [ ] Timer and message correlation expressions
- [ ] DMN input expressions, input entries, output entries, and literal expressions

### Syntax and Prefixes
- [ ] BPMN FEEL expressions that require expression mode start with `=`
- [ ] String literals use quotes consistently
- [ ] Boolean comparisons use booleans, not string values like `"true"`
- [ ] List indexes and filters match FEEL semantics
- [ ] Context and property access matches the expected result shape

### Variable Availability
- [ ] Every referenced variable is written before the expression can run
- [ ] Variable casing matches exactly across BPMN, DMN, and worker mappings
- [ ] Nested property access handles absent parent objects where needed
- [ ] Expressions do not reference variables local to another scope

### Null and Type Safety
- [ ] Numeric comparisons guard against `null` when the variable may be absent
- [ ] Date/time comparisons use compatible types
- [ ] List operations handle empty lists
- [ ] DMN rules have clear behavior for null input values

## Reporting Format

```
## [filename.bpmn]

### FEEL Expressions
- ❌ "Order Approved?" — `= orderAmount > 100` can fail when `orderAmount` is null on the manual-review path
- ⚠️  "Has Discount?" — compares `hasDiscount == "true"`; confirm this is a string and not a boolean
- ✅ "Extract Decision" — `= decisionResult.eligible` matches the DMN result shape
```

## Instructions

1. Locate target BPMN and DMN files using the path argument or by searching the working directory.
2. Extract all FEEL expressions from sequence flows, mappings, event definitions, and DMN elements.
3. Parse variable references and cross-reference them with known write points and scopes.
4. Review expression syntax, null handling, and likely type mismatches.
5. Report findings grouped by file and expression location.
6. Summarise: total expressions checked, pass count, warning count, failure count.
