# Process Versioning Audit

Audits process, decision, form, and call activity references for deployment-time versioning risks. Use this before deploying multiple related BPMN and DMN artifacts.

## Usage

```
/process-versioning-audit [file-or-directory]
```

If no path is given, scan all BPMN, DMN, and form files in the current directory tree.

## What This Checks

### Process IDs
- [ ] Every BPMN process has a non-empty `id`
- [ ] Process IDs are unique across the deployment package
- [ ] Process ID casing matches call activity references exactly
- [ ] Process names are descriptive but not used as deployment keys

### Call Activity Binding
- [ ] Every call activity references an existing process ID or an explicitly external deployed process
- [ ] Binding mode is intentional: latest, deployment, version, or versionTag
- [ ] Version or versionTag references exist and are documented when pinned
- [ ] Child process changes are reviewed with all parent processes that call them

### Decision References
- [ ] Every business rule task decision reference matches a DMN decision ID
- [ ] Decision IDs are unique across deployed DMN files
- [ ] Decision versioning or deployment grouping is intentional
- [ ] DRD dependencies are deployed together when required

### Form References
- [ ] User task form references resolve to deployed or repository-local forms
- [ ] Form versions are compatible with process expectations
- [ ] Form field changes are reviewed against downstream variable usage

### Deployment Package
- [ ] BPMN, DMN, and form files that depend on each other are deployed together
- [ ] No duplicate artifact IDs will cause accidental latest-version selection
- [ ] Environment-specific IDs, names, or version tags are avoided unless documented

## Reporting Format

```
## Deployment Package

### Versioning Findings
- ❌ `order-fulfillment` — duplicate process ID found in order-v1.bpmn and order-v2.bpmn
- ❌ "Process Payment" — call activity references `payment-processing`, but no matching process ID was found
- ⚠️  "Calculate Discount" — decision reference uses latest deployed decision; confirm this is intentional
- ✅ `shippingEligibility` — decision ID is unique and deployed with caller
```

## Instructions

1. Locate BPMN, DMN, and form files using the path argument or by searching the working directory.
2. Build an index of process IDs, decision IDs, form IDs, and deployment artifacts.
3. Resolve call activity, business rule task, and user task references against the index.
4. Identify duplicate IDs, missing references, risky latest-version assumptions, and deployment grouping gaps.
5. Report findings grouped by artifact type and file.
6. Summarise: total artifacts checked, pass count, warning count, failure count.
