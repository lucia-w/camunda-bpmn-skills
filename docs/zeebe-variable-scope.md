# Zeebe Variable Scope

How variables flow through Zeebe process instances, and where scope isolation can surprise you.

## The Core Rule

Every process instance has its own variable scope. Call activities can automatically copy variables across the parent/child boundary, but relying on that default is fragile. The safest production pattern is to treat every call activity like a function call: explicitly map the inputs the child needs and explicitly map the outputs the parent needs.

## Variable Scope by Element Type

### Service Tasks
Variables written via `zeebe:output` are written into the **local scope of the enclosing process instance** — not globally. If the service task is inside a sub-process (embedded), variables are written to the sub-process scope and are visible within it and all parent scopes of that same instance.

### Call Activities
```xml
<callActivity id="createVaultEntry" calledElement="vault-creation">
  <extensionElements>
    <zeebe:calledElement
      propagateAllParentVariables="false"
      propagateAllChildVariables="false" />
    <zeebe:input source="= customerId" target="customerId" />
    <zeebe:input source="= loanAmount" target="loanAmount" />
    <zeebe:output source="= vaultId" target="vaultId" />
  </extensionElements>
</callActivity>
```

- `zeebe:input` copies a variable from the **parent** scope into the **child** scope at start
- `zeebe:output` copies a variable from the **child** scope into the **parent** scope at completion
- If broad propagation is disabled, only explicit mappings cross the boundary

### `propagateAllParentVariables`

In Camunda 8, `propagateAllParentVariables` defaults to `true`: variables visible in the call activity scope are copied into the created child process instance unless you disable propagation.

| Value | Effect |
|-------|--------|
| `true` | All parent variables are copied into the child scope at start — child can read everything the parent has |
| `false` | Only explicitly mapped `zeebe:input` variables are available in the child scope |

`propagateAllChildVariables` also defaults to `true`: variables from the completed child process are copied back to the call activity unless you disable propagation or define output mappings. In production models, prefer `propagateAllChildVariables="false"` with explicit `zeebe:output` mappings so child-internal variables do not pollute or overwrite the parent scope.

### Embedded Sub-Processes
An embedded `<subProcess>` shares scope with its parent process instance. Variables set inside are visible outside without any mapping. This is different from call activities.

### Multi-Instance Sub-Processes
Each loop instance gets its own local variable scope. The loop input element (`inputElement`) is the variable available inside each iteration. Variables set inside one iteration are **not** visible to other iterations.

## Common Scope Bugs

### The Scope Trap
A variable is set inside a call activity and the developer assumes it is available in the parent after the call activity completes.

```
Parent process
  -> Call Activity "Create Vault Entry"
      -> Service Task "Store Vault Record" sets vaultId via zeebe:output
  <- Call Activity completes
  -> Service Task "Notify Vault System" reads vaultId  <- NULL if child variables are not propagated or mapped out
```

Fix: disable implicit child propagation and add `<zeebe:output source="= vaultId" target="vaultId" />` to the call activity.

### The Shadow Variable
A variable name is reused in both parent and child. The child mapping overwrites the parent's copy when the child completes, or vice versa.

Fix: use distinct variable names across process boundaries, or explicitly rename via the `target` attribute in `zeebe:output`.

### The Inherited Poison
`propagateAllParentVariables=true` means the child process sees a variable from the parent and uses it — but the parent later changes that variable (in a parallel branch), and the child's snapshot is now stale.

Fix: always map only what the child needs explicitly.

## Debugging Scope Issues

Use `/variable-trace <variableName>` to see every write and read point and where scope boundaries exist.

In Operate (Camunda's UI), inspect the variable tab at each element level — variables are shown in the scope where they live, which makes scope isolation bugs immediately visible.
