---
description: Validate FEEL expressions extracted from Camunda 8 BPMN and DMN files
allowed-tools: Read, Bash, Glob, Grep
argument-hint: <path/to/file.bpmn or file.dmn or directory>
---

Extract and validate all FEEL (Friendly Enough Expression Language) expressions from the Camunda 8 BPMN and DMN file(s) at `$ARGUMENTS`. If no argument is given, scan the entire project for `*.bpmn` and `*.dmn` files.

## What is FEEL?

FEEL is the expression language used in Camunda 8 for:
- Sequence flow conditions (`conditionExpression`)
- Variable input/output mappings (`zeebe:input source`, `zeebe:output source`)
- Timer expressions (`timeDuration`, `timeCycle`, `timeDate`)
- Message correlation keys (`zeebe:subscription correlationKey`)
- User task assignment (`assignee`, `candidateGroups`)
- DMN input entries, output entries, and input expressions
- Literal expressions in DMN decisions
- Script task expressions

## Steps

### 1. Extract all FEEL expressions

Collect every FEEL expression location. For each, record:
- File name and line number
- Element type and element ID
- Attribute or child element name
- Raw expression text

Locations to check in BPMN:
```xml
<conditionExpression>expression</conditionExpression>
<zeebe:input source="expression" />
<zeebe:output source="expression" />
<zeebe:subscription correlationKey="expression" />
<zeebe:calledElement processId="=expression" />
<zeebe:calledDecision decisionId="=expression" />
<zeebe:assignmentDefinition assignee="expression" />
<zeebe:assignmentDefinition candidateGroups="expression" />
<timeDuration>expression</timeDuration>
<timeCycle>expression</timeCycle>
<timeDate>expression</timeDate>
```

Locations to check in DMN:
```xml
<inputExpression><text>expression</text></inputExpression>
<inputEntry><text>expression</text></inputEntry>
<outputEntry><text>expression</text></outputEntry>
<literalExpression><text>expression</text></literalExpression>
```

### 2. Classify each expression

Determine the expression's **context type**, which governs the valid FEEL syntax:

| Context | Expected FEEL type | Notes |
|---|---|---|
| `conditionExpression` | Unary test or expression returning boolean | Should return `true`/`false` |
| `zeebe:input source` | Expression (value) | Variable path or `=` expression |
| `zeebe:output source` | Expression (value) | Variable path or `=` expression |
| `correlationKey` | Expression (value) | Must evaluate to a string or number |
| `timeDuration` | Duration literal or `=` expression | ISO 8601 or FEEL |
| `timeCycle` | Cycle literal or `=` expression | ISO 8601 or FEEL |
| `timeDate` | Date-time literal or `=` expression | ISO 8601 or FEEL |
| `inputEntry` (DMN) | Unary test | e.g., `> 100`, `"active"`, `[1..5]` |
| `outputEntry` (DMN) | Expression (value) | e.g., `"approved"`, `42`, `applicant.name` |
| `inputExpression` (DMN) | Expression (path) | e.g., `order.amount` |
| `literalExpression` (DMN) | Full FEEL expression | Entire expression returning a value |

### 3. Validate syntax rules

Apply these checks to every expression:

#### 3a. String literals
- String values must be enclosed in double quotes: `"hello"`, not `hello` or `'hello'`.
- Flag unquoted bare words that are not valid FEEL identifiers or keywords.
- Flag single-quoted strings (not valid in FEEL).

#### 3b. Numeric literals
- Plain numbers are valid: `42`, `3.14`, `-7`.
- Flag numbers with commas as thousand separators: `1,000` — this is a disjunction in FEEL, not a number.

#### 3c. Boolean literals
- Valid: `true`, `false`.
- Flag quoted booleans: `"true"` in an input that expects a boolean type — this is a string, not a boolean.

#### 3d. Null
- `null` is a valid FEEL literal.
- Flag uses of `nil`, `undefined`, `None` — these are not FEEL.

#### 3e. Arithmetic operators
- Valid: `+`, `-`, `*`, `/`, `**` (exponentiation).
- Flag `%` (modulo is not a FEEL operator; use `modulo(a, b)` function instead).
- Flag `^` for exponentiation (use `**`).

#### 3f. Comparison operators
- Valid: `=`, `!=`, `<`, `<=`, `>`, `>=`.
- Flag `==` — this is not FEEL; use `=` for equality.
- Flag `!` as a prefix negation — use `not(expression)`.

#### 3g. Logical operators
- Valid: `and`, `or`.
- Flag `&&`, `||`, `!` — these are not FEEL.

#### 3h. Range expressions
- Valid: `[1..10]` (inclusive-inclusive), `(0..100)` (exclusive-exclusive), `[0..100)`, `(0..100]`.
- Flag ranges that mix `..` with `-`: `[1-10]` — this is subtraction, not a range.
- Flag ranges without `..`: `[1,10]` — this is a list, not a range.

#### 3i. Unary tests (DMN input entries)
- Unary tests are tested against an implicit input value.
- Valid forms: `> 5`, `< 10`, `[1..100]`, `"active"`, `"a", "b", "c"`, `not("cancelled")`, `-` (don't care / match all).
- Flag `= "active"` in unary test context — in DMN input entries, equality is expressed without `=`: just `"active"`.
- Flag `true = someVar` — in unary tests, the implicit value is on the left; comparing to a variable requires `= someVar`.

#### 3j. Path expressions
- Valid: `order.item.price`, `customer.address.city`.
- Flag bracket notation: `order["item"]` — this is filter/access syntax; valid only in certain contexts.
- Flag snake_case confusion: `order_total` is a valid identifier; `order.total` is a path. Both are valid but different.

#### 3k. Function calls
- Common built-in FEEL functions: `count()`, `sum()`, `min()`, `max()`, `mean()`, `all()`, `any()`, `sublist()`, `append()`, `concatenate()`, `insert before()`, `remove()`, `reverse()`, `index of()`, `union()`, `distinct values()`, `flatten()`, `product()`, `median()`, `stddev()`, `mode()`, `abs()`, `ceiling()`, `floor()`, `round up()`, `round down()`, `round half up()`, `round half down()`, `decimal()`, `sqrt()`, `log()`, `exp()`, `odd()`, `even()`, `before()`, `after()`, `meets()`, `met by()`, `overlaps()`, `overlaps before()`, `overlaps after()`, `finishes()`, `finished by()`, `includes()`, `during()`, `starts()`, `started by()`, `coincides()`, `day of week()`, `day of year()`, `week of year()`, `month of year()`, `now()`, `today()`, `string()`, `number()`, `boolean()`, `date()`, `date and time()`, `time()`, `duration()`, `years and months duration()`, `contains()`, `starts with()`, `ends with()`, `matches()`, `replace()`, `split()`, `substring()`, `string length()`, `upper case()`, `lower case()`, `substring before()`, `substring after()`, `list contains()`, `not()`, `get value()`, `get entries()`, `context()`, `context merge()`, `context put()`.
- Flag calls to functions not in this list (they may be custom or misspelled).
- Flag `length()` (not FEEL; use `count()` for lists and `string length()` for strings).

#### 3l. Date and time literals
- Date: `date("2025-01-15")` — string argument in double quotes.
- Time: `time("14:30:00")` or `time("14:30:00+01:00")`.
- Date-time: `date and time("2025-01-15T14:30:00")`.
- Duration: `duration("P1D")`, `duration("PT2H30M")`.
- Flag bare date strings not wrapped in a function: `2025-01-15` (interpreted as `2025 - 1 - 15 = 2009`, not a date).

#### 3m. Context and list literals
- Context: `{key: value, key2: value2}` — keys are unquoted names.
- List: `[1, 2, 3]`, `["a", "b"]`.
- Flag JavaScript-style objects: `{"key": "value"}` — FEEL context keys are not quoted.

#### 3n. if/then/else
- Valid: `if condition then value1 else value2`.
- Flag ternary `condition ? value1 : value2` — not valid FEEL.

#### 3o. for/some/every/satisfies
- Valid: `for i in list return i * 2`.
- Valid: `some x in list satisfies x > 0`.
- Valid: `every x in list satisfies x > 0`.
- Flag JavaScript-style `list.map()` or `list.filter()` — not FEEL.

#### 3p. `=` prefix (Camunda expression mode)
- In Camunda 8, a FEEL expression that starts with `=` is evaluated as a full expression.
- Without `=`, most contexts treat the text as a literal value or unary test.
- Flag `conditionExpression` values that do NOT start with `=` and are not a valid unary test — they will be treated as string literals and always evaluate to `true`.

### 4. Variable reference analysis

For each FEEL expression that references variables:
- Collect all variable names referenced (top-level identifiers that are not FEEL keywords or built-in function names).
- Cross-reference with variables that are:
  - Defined as process variables at the start event (look for initial variable definitions in BPMN).
  - Set by preceding output mappings or task results in the same process path.
  - Defined as the `resultVariable` of a business rule task or call activity.
- Flag variables that appear in expressions but have no visible definition in the process — they may be set externally, but unresolvable references are worth noting.

### 5. Timer expression validation

- `timeDuration` / `timeCycle` / `timeDate` without `=` prefix: must be a valid ISO 8601 string.
  - Duration: matches regex `^P(\d+Y)?(\d+M)?(\d+W)?(\d+D)?(T(\d+H)?(\d+M)?(\d+(\.\d+)?S)?)?$`
  - Repeating: `R<n>/PT<duration>` or `R/PT<duration>` (infinite repeat).
  - Date-time: `YYYY-MM-DDTHH:MM:SSZ` or with offset.
- With `=` prefix: must be a valid FEEL expression returning the appropriate type.
- Flag completely invalid strings like `"every hour"`, `"1d"`, `"24h"`.

## Output format

```
## FEEL Validation Report

### Expressions Scanned: <N>

### ❌ Errors (will cause runtime failure or wrong behavior)
| File | Line | Element | Attribute | Expression | Issue |
|---|---|---|---|---|---|
| order.bpmn | 42 | SequenceFlow_1 | conditionExpression | `amount > 100` | Missing `=` prefix — treated as string literal, always true |
| rules.dmn | 18 | rule[3] inputEntry[1] | — | `pending` | Unquoted string — must be `"pending"` |

### ⚠️ Warnings (valid but potentially wrong)
| File | Line | Element | Attribute | Expression | Issue |
|---|---|---|---|---|---|
| order.bpmn | 67 | Task_1 | zeebe:output source | `=result.amount` | Variable `result` not visible in preceding flow — may be undefined |

### ✅ Valid expressions: <N>

### 📋 Summary
- Total expressions: N
- Errors: A  Warnings: B  Valid: C
- Verdict: ALL FEEL OK / FEEL ISSUES FOUND
```
