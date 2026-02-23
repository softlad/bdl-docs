# Business Decision Language (BDL) — Technical Specification

**Version:** 1.1
**Status:** Authoritative reference for BDL document grammar, execution, and integration.
**Supersedes:** Version 1.0

---

## Changelog: v1.0 → v1.1

| Section | Change |
|---------|--------|
| §2 | `extends` and `params` added to BDL_Document |
| §4 | Temporal predicates added to Predicate grammar |
| §9 | Parameterised policy evaluation semantics added |
| §10 | Policy composition semantics added |
| §11 | Hard constraints updated for new features |
| §18 | **New** — Parameterised Policies |
| §19 | **New** — Policy Composition |
| §20 | **New** — Test Case Format |

All constructs from v1.0 remain valid and unchanged. v1.1 is fully backwards compatible.

---

## 1. Conceptual Model

### 1.1 Case

A **Case** is an immutable JSON object supplied at runtime. It represents a single decision instance. Field paths use dot notation (e.g. `expense.amount`, `request.item`). The shape of a Case is defined by the policy's generated schema; no fixed global schema is required.

### 1.2 BDL Document (Policy)

A **BDL document** is a single YAML or JSON artifact that contains:

- Document-level metadata (policy_id, version, effective dates, jurisdiction, defaults)
- Optional **params** — declared parameters that can be supplied at runtime to make the policy reusable across contexts *(v1.1)*
- Optional **extends** — a reference to a base policy document whose statements are inherited *(v1.1)*
- Optional **tables** — lookup tables for lookup-based decisioning (see §2, TableDefinition)
- An ordered list of **statements**

A **Policy Package** is a versioned bundle that may include one or more BDL documents, a generated input schema, lookup tables, and metadata. **Lookup-based decisioning** is when an output (e.g. price, rate, tier) is determined by matching the Case to a table row via a composite key; the Value grammar's **Lookup** and DEFINE rules use these tables.

### 1.3 Statement

A **statement** is the atomic unit of logic. Each statement has:

- **id** — unique identifier
- **type** — one of DEFINE, REQUIRE, ALLOW, FORBID, LIMIT, ROUTE, TAG
- **priority** — integer used for precedence (higher wins)
- **applies_when** — optional predicate; when absent the statement always applies
- **rule** — type-specific payload
- **outcomes** — what to report on apply, violation, missing data, or error
- **cite** — optional list of citations to source documents

---

## 2. Document Grammar

Top-level structure of a BDL document:

```
BDL_Document ::= {
  ir_version: string,           // e.g. "1.1"
  policy_id: string,
  policy_name?: string,
  version: string,
  effective: { start: date, end?: date },
  jurisdiction?: string[],
  priority_model: "explicit",
  defaults: Defaults,
  params?: ParamDefinition[],   // v1.1: declared runtime parameters
  extends?: PolicyRef,          // v1.1: base policy to inherit from
  tables?: TableDefinition[],
  statements: Statement[]
}
```

- **defaults**: e.g. `{ on_missing: needs_info | needs_review, on_error: needs_review }`
- **params**: optional array of parameter declarations. See §18.
- **extends**: optional reference to a base policy. See §19.
- **tables**: optional array of lookup tables. See TableDefinition below.
- Only the fields listed are permitted at the top level.

### 2.1 TableDefinition (lookup tables)

```
TableDefinition ::= {
  id: string,                   // unique; referenced by Lookup.table
  key_columns: string[],        // column names forming composite key (order matches Lookup.key)
  value_column: string,         // column name for the output value
  rows: Row[]
}

Row ::= object                  // keys = key_columns ∪ { value_column }; values = literals
```

At runtime, a **Lookup** value supplies `table` (id) and `key` (array of FieldRefs). The engine reads the Case (and DEFINE-derived context) at each key path, finds the row whose key_columns match those values in order, and returns that row's value_column. No match → implementation may treat as missing or apply on_error/defaults.

---

## 3. Statement Grammar

```
Statement ::= {
  id: string,
  type: StatementType,
  priority: integer,
  applies_when?: Predicate,
  rule: RuleBody,
  outcomes: OutcomeBlock,
  cite: Citation[],
  meta?: Meta
}

StatementType ::= DEFINE | REQUIRE | ALLOW | FORBID | LIMIT | ROUTE | TAG
```

No other statement types are permitted.

---

## 4. Predicate Grammar

Predicates are declarative boolean expressions over the Case (and any DEFINE-derived context).

```
Predicate ::=
  { all: Predicate[] }
| { any: Predicate[] }
| { not: Predicate }
| Comparison

Comparison ::=
  { eq: [FieldRef, Value] }
| { neq: [FieldRef, Value] }
| { lt: [FieldRef, Value] }
| { lte: [FieldRef, Value] }
| { gt: [FieldRef, Value] }
| { gte: [FieldRef, Value] }
| { in: [FieldRef, Value[]] }
| { exists: [FieldRef] }
| { contains: [FieldRef, Value] }
| TemporalComparison                 // v1.1

FieldRef ::= string   // dot-notation path, e.g. "expense.amount"
```

- **eq / neq**: equality (value may be string, number, boolean, null).
- **lt / lte / gt / gte**: numeric comparison; the value in the Case at FieldRef is compared to the given Value.
- **in**: value at FieldRef must be in the given array.
- **exists**: value at FieldRef is not undefined and not null.
- **contains**: for arrays, array includes Value; for strings, string contains Value (string).

### 4.1 Temporal Comparisons *(v1.1)*

```
TemporalComparison ::=
  { before: [FieldRef, TemporalValue] }
| { after: [FieldRef, TemporalValue] }
| { within: [FieldRef, Duration] }
| { elapsed: [FieldRef, Duration] }

TemporalValue ::=
  DateLiteral                         // ISO 8601 date string, e.g. "2025-01-01"
| DateTimeLiteral                     // ISO 8601 datetime, e.g. "2025-01-01T00:00:00Z"
| { now: true }                       // runtime clock at evaluation time
| { param: string }                   // reference to a declared param (see §18)
| { field: FieldRef }                 // another date/datetime field in the Case

Duration ::= {
  value: number,
  unit: "minutes" | "hours" | "days" | "weeks" | "months" | "years"
}
```

**Semantics:**

- **before**: the date/datetime at FieldRef is strictly earlier than TemporalValue.
- **after**: the date/datetime at FieldRef is strictly later than TemporalValue.
- **within**: the date/datetime at FieldRef is no more than Duration before `now`. Equivalent to `after: [field, { now - duration }]`. Useful for recency checks (e.g. "submitted within the last 30 days").
- **elapsed**: the date/datetime at FieldRef is at least Duration before `now`. Equivalent to `before: [field, { now - duration }]`. Useful for age or advance-booking checks (e.g. "booked at least 14 days ago").

**Evaluation rules:**

- FieldRef must resolve to a date or datetime value. If absent → `on_missing`. If present but not parseable as a date → `on_error`.
- `{ now: true }` is resolved once per evaluation run and held constant for the duration of that run. It is recorded in the trace.
- Month-based durations (months, years) are calendar-aware (e.g. one month after January 31 is February 28/29, not March 2/3).
- All comparisons are timezone-aware. Date-only values (no time component) are treated as midnight UTC.

**Example — advance booking check:**

```yaml
applies_when:
  elapsed:
    - travel.booking_date
    - value: 14
      unit: days
```

**Example — recent submission check:**

```yaml
applies_when:
  within:
    - expense.submitted_at
    - value: 30
      unit: days
```

**Example — effective date check:**

```yaml
applies_when:
  all:
    - after: [contract.start_date, "2024-01-01"]
    - before: [contract.start_date, { param: cutoff_date }]
```

---

## 5. Value Grammar

```
Value ::= Literal | Lookup | Arithmetic

Literal ::= string | number | boolean | null

Lookup ::= {
  lookup: {
    table: string,
    key: FieldRef[]
  }
}

Arithmetic ::=
  { add: Value[] }
| { sub: Value[] }
| { mul: Value[] }
| { div: Value[] }
```

- **Lookup**: used for **lookup-based decisioning**. `table` names a table in the document's `tables` (see §2.1). `key` is an array of FieldRefs; the engine reads the Case (and DEFINE context) at each path, in order, to form the composite key. The row in that table whose key_columns equal those values is selected; the result is that row's value_column. Commonly used in DEFINE to set an output (e.g. output.unit_price) from a table.
- **Arithmetic**: must be side-effect free and finite.
- **Param references in Values**: any Value position may use `{ param: "name" }` to reference a declared parameter (see §18). The param value is substituted before evaluation.

---

## 6. Rule Bodies by Statement Type

### DEFINE

```
rule ::= {
  set: { target: FieldRef, value: Value }[]
}
```

Derives or normalizes data; results are available to subsequent statements.

### REQUIRE

```
rule ::= {
  require_fields?: FieldRef[],
  require_evidence?: string[]
}
```

- **require_fields**: listed paths must be present and non-missing in the Case (or derived context).
- **require_evidence**: listed evidence identifiers must be present in the Case (e.g. in an evidence array).

### ALLOW / FORBID

```
rule ::= {
  field: FieldRef,
  values: Value[]
}
```

- **ALLOW**: when applies_when is true and the value at `field` is in `values`, the statement's on_apply outcome is used.
- **FORBID**: when applies_when is true and the value at `field` is in `values`, the statement's on_violation outcome is used.

### LIMIT

```
rule ::= {
  field: FieldRef,
  op: "lt" | "lte" | "gt" | "gte",
  value: Value
}
```

The value at `field` is compared to `value` using `op`. If the comparison fails, on_violation is used.

### ROUTE

```
rule ::= {
  to: string,
  sla_hours?: number
}
```

`to` is a destination identifier (e.g. approval queue, role). Typically used with on_apply → verdict needs_review.

### TAG

```
rule ::= {
  add: string[]
}
```

Adds classification labels to the result (e.g. for audit or downstream routing). Often used with verdict no_change.

---

## 7. Outcome Grammar

```
OutcomeBlock ::= {
  on_apply?: Outcome,
  on_violation?: Outcome,
  on_missing?: Outcome,
  on_error?: Outcome
}

Outcome ::= {
  verdict: Verdict,
  reason_code?: string,
  severity?: "low" | "medium" | "high",
  override?: boolean,
  halt?: boolean
}

Verdict ::= compliant | non_compliant | needs_info | needs_review | no_change
```

- **on_apply**: when the statement's condition and rule are satisfied (e.g. ALLOW matches, REQUIRE satisfied).
- **on_violation**: when the rule is violated (e.g. FORBID matches, LIMIT exceeded).
- **on_missing**: when required fields or evidence are missing.
- **on_error**: when evaluation of the statement throws or fails.

**override**: when true, this outcome can override a lower-priority statement's outcome.
**halt**: when true, evaluation may stop (implementation-defined).

### 7.1 Verdicts

| Verdict | Meaning | Use |
|--------|--------|-----|
| **compliant** | The case satisfies the rule. | Used in **on_apply** when the rule passes. |
| **non_compliant** | The case violates the rule. | Used in **on_violation** when the rule fails. |
| **needs_info** | Required data or evidence is missing. | Used in **on_missing** when fields or evidence are absent. |
| **needs_review** | The case requires human or downstream review. | Used in **on_apply** (e.g. ROUTE), **on_missing**, or **on_error**. |
| **no_change** | No effect on the overall outcome. | Used when the statement only tags or annotates. |

---

## 8. Citation Grammar

```
Citation ::= {
  doc_id: string,
  section?: string,
  clause_id?: string,
  span?: { start: number, end: number },
  hash?: string
}
```

---

## 9. Defaults and Evaluation Semantics

### 9.1 Defaults

Document-level `defaults` apply when a statement does not specify an outcome for a given situation:

- **on_missing**: default verdict when required data is missing (e.g. needs_info, needs_review).
- **on_error**: default verdict when evaluation fails (e.g. needs_review).

### 9.2 Execution Order

1. **DEFINE** statements run first (order may be defined by dependency or document order).
2. All other statements are evaluated in **descending priority** (higher number first).
3. When multiple statements fire, the **highest-priority** outcome wins unless a higher-priority statement has **override: true** and fires later in the sort order.

### 9.3 Missing Data

- Missing required fields or evidence does not imply a value; it triggers on_missing (or document default).
- Verdicts **needs_info** and **needs_review** are first-class; no implicit assumptions fill missing data.

### 9.4 Trace

Every evaluation produces a **trace** that includes:

- Final verdict and reason codes
- Required fields (if any) still missing
- Which statements were evaluated and with what result (applied, violation, missing, skipped)
- Effective execution profile (if supplied)
- Citations for fired statements
- Resolved value of `{ now: true }` if any temporal predicate was evaluated *(v1.1)*
- Resolved param values used during evaluation *(v1.1)*
- The `policy_id` and `version` of the base policy, if `extends` was used *(v1.1)*

### 9.5 Param Resolution *(v1.1)*

Before evaluation begins, the engine resolves all declared params:

1. Each param declared in `params` is matched against the `params` object supplied in the ExecutionRequest.
2. If a required param is absent and has no `default`, evaluation halts with `on_error` (document default applies).
3. Resolved param values are substituted wherever `{ param: "name" }` appears in predicates and values.
4. All resolved param values are recorded in the trace.

### 9.6 Composition Resolution *(v1.1)*

When a document declares `extends`, the engine resolves the full statement set before evaluation:

1. The base policy identified by `extends` is loaded at the version specified.
2. Statements from the base policy are merged with the extending document's statements into a single ordered set.
3. If a statement in the extending document shares an `id` with a base statement, the extending document's version **overrides** the base. This is the mechanism for specialisation.
4. Merged statements are then evaluated according to §9.2.
5. The trace records the origin of each fired statement (base or extending document).

---

## 10. Execution Profiles

Execution scope is controlled by a **profile** supplied at runtime with the Case. The profile is **not** part of the BDL document.

### 10.1 Execution Request

```
ExecutionRequest ::= {
  profile?: ExecutionProfile,
  case: object,
  params?: object,              // v1.1: runtime param values (see §18)
  policy_id?: string,
  version?: string
}
```

If `profile` is omitted, the runtime behaves as for full enforcement.

### 10.2 ExecutionProfile

```
ExecutionProfile ::= {
  evaluate_types: StatementType[],
  missing_data_behavior?: "enforce" | "ask" | "ignore"
}
```

### 10.3 Reference Profiles (Non-Normative)

| Profile name | evaluate_types | missing_data_behavior |
|---|---|---|
| ADVISORY_PERMISSIBILITY | DEFINE, ALLOW, FORBID, TAG | ignore |
| CONSTRAINT_CHECK | DEFINE, ALLOW, FORBID, LIMIT, TAG | ask |
| FULL_ENFORCEMENT | DEFINE, ALLOW, FORBID, LIMIT, REQUIRE, ROUTE, TAG | enforce |

---

## 11. Hard Constraints (Compiler / Authoring)

When generating or authoring BDL:

1. Use only constructs defined in this specification.
2. Do not introduce new statement types, comparison operators, or verdicts.
3. Every statement should include at least one citation.
4. Represent ambiguity via needs_info or needs_review outcomes, not by omitting outcomes.
5. Do not encode business meaning outside statements (e.g. in comments only).
6. All behavior must be explainable via the trace.
7. All params referenced in predicates or values must be declared in `params`. *(v1.1)*
8. Param `default` values must satisfy the declared `type`. *(v1.1)*
9. Circular `extends` chains are not permitted; the compiler must detect and reject them. *(v1.1)*
10. A statement `id` used in an extending document to override a base statement must be explicitly marked `override: true` at the statement level to make the intent clear. *(v1.1)*
11. Temporal FieldRefs must resolve to date or datetime values; the compiler should warn where the declared schema type is ambiguous. *(v1.1)*

---

## 12–17. (Unchanged from v1.0)

*Sections 12 through 17 (examples and MCP integration) are unchanged from v1.0 and are incorporated by reference.*

---

## 18. Parameterised Policies *(v1.1)*

Parameters allow a BDL document to declare runtime-configurable values, making a single policy reusable across contexts that share the same logic but differ in thresholds, identifiers, or dates.

### 18.1 ParamDefinition

```
ParamDefinition ::= {
  name: string,                        // unique within the document
  type: "string" | "number" | "boolean" | "date" | "datetime",
  required: boolean,
  default?: Literal,                   // must match type; omit if required: true
  description?: string
}
```

- `name` must be unique within the document's `params` array.
- `required: true` means the caller must supply the param in the ExecutionRequest. If absent and no `default` is declared, evaluation halts.
- `required: false` with a `default` means the default is used when the caller does not supply the param.

### 18.2 Param References

Params are referenced in predicates and values using:

```
{ param: "name" }
```

This syntax is valid anywhere a `TemporalValue` or `Value` (including `Literal`) is accepted. The engine substitutes the resolved param value before evaluation.

### 18.3 Example

**Policy:** Flag expenses submitted after a configurable cutoff date.

```yaml
ir_version: "1.1"
policy_id: expense_cutoff_policy
version: "1.0.0"
effective:
  start: "2025-01-01"
defaults:
  on_missing: needs_info
  on_error: needs_review

params:
  - name: submission_cutoff
    type: date
    required: true
    description: "Expenses submitted after this date are flagged for review."

statements:
  - id: FLAG_LATE_SUBMISSION
    type: FORBID
    priority: 70
    applies_when:
      after:
        - expense.submitted_date
        - param: submission_cutoff
    rule:
      field: expense.submitted_date
      values: []
    outcomes:
      on_violation:
        verdict: needs_review
        reason_code: SUBMISSION_AFTER_CUTOFF
    cite:
      - doc_id: EXPENSE_POLICY_2025
        section: "4.2"
```

**ExecutionRequest:**

```json
{
  "params": { "submission_cutoff": "2025-03-31" },
  "case": {
    "expense": { "submitted_date": "2025-04-05", "amount": 120 }
  }
}
```

The same policy document can be reused for different cutoff periods by varying the `params` object at runtime.

---

## 19. Policy Composition *(v1.1)*

Policy composition allows a BDL document to inherit statements from a base policy, then extend or override them. This supports organisational patterns where a global base policy is shared and jurisdiction-specific or team-specific documents add rules without duplication.

### 19.1 PolicyRef

```
PolicyRef ::= {
  policy_id: string,
  version: string             // exact version pin; ranges are not permitted
}
```

A document may declare at most one `extends`. Chained inheritance (A extends B extends C) is permitted; circular references are not.

### 19.2 Statement Merging

The engine constructs a merged statement set as follows:

1. Load the base policy (recursively resolving its own `extends` if present).
2. Collect all statements from the base.
3. For each statement in the extending document:
   - If its `id` does not exist in the base: **add** it to the merged set.
   - If its `id` exists in the base and the statement is marked `override: true`: **replace** the base statement.
   - If its `id` exists in the base but `override` is not set: **compiler error** — ambiguous intent.
4. Evaluate the merged set according to §9.2.

### 19.3 Params in Composed Policies

- The base policy's `params` declarations are inherited.
- The extending document may declare additional params.
- The extending document must not redeclare a param name already declared in the base (compiler error).
- At runtime, a single `params` object is supplied; it must satisfy the union of all declared params.

### 19.4 Example

**Base policy** (`global_expense_policy` v1.0.0):

```yaml
ir_version: "1.1"
policy_id: global_expense_policy
version: "1.0.0"
effective:
  start: "2025-01-01"
defaults:
  on_missing: needs_info
  on_error: needs_review

params:
  - name: meal_limit
    type: number
    required: false
    default: 25

statements:
  - id: MEAL_REQUIRE_RECEIPT
    type: REQUIRE
    priority: 80
    applies_when:
      all:
        - eq: [expense.category, MEAL]
        - gt: [expense.amount, { param: meal_limit }]
    rule:
      require_evidence: [ITEMIZED_RECEIPT]
    outcomes:
      on_apply:
        verdict: compliant
        reason_code: RECEIPT_MEETS_REQUIREMENT
      on_missing:
        verdict: needs_review
        reason_code: ITEMIZATION_REQUIRED
    cite:
      - doc_id: GLOBAL_EXPENSE_POLICY
        section: "3.1"
```

**Extending policy** (`uk_expense_policy` v1.0.0):

```yaml
ir_version: "1.1"
policy_id: uk_expense_policy
version: "1.0.0"
effective:
  start: "2025-01-01"
jurisdiction: [GB]

extends:
  policy_id: global_expense_policy
  version: "1.0.0"

defaults:
  on_missing: needs_info
  on_error: needs_review

params:
  - name: vat_receipt_required
    type: boolean
    required: false
    default: true

statements:
  - id: MEAL_REQUIRE_RECEIPT
    override: true              # replaces the base statement
    type: REQUIRE
    priority: 80
    applies_when:
      all:
        - eq: [expense.category, MEAL]
        - gt: [expense.amount, { param: meal_limit }]
    rule:
      require_evidence: [ITEMIZED_RECEIPT, VAT_RECEIPT]
    outcomes:
      on_apply:
        verdict: compliant
        reason_code: RECEIPT_MEETS_UK_REQUIREMENT
      on_missing:
        verdict: needs_review
        reason_code: UK_ITEMIZATION_REQUIRED
    cite:
      - doc_id: UK_EXPENSE_POLICY
        section: "2.4"

  - id: UK_MILEAGE_LIMIT
    type: LIMIT
    priority: 75
    applies_when:
      eq: [expense.category, MILEAGE]
    rule:
      field: expense.rate_per_mile
      op: lte
      value: 0.45
    outcomes:
      on_violation:
        verdict: non_compliant
        reason_code: MILEAGE_RATE_EXCEEDS_HMRC_LIMIT
    cite:
      - doc_id: HMRC_MILEAGE_RATES
        section: "2025"
```

The merged evaluation set contains: the overridden `MEAL_REQUIRE_RECEIPT` (UK version) plus the new `UK_MILEAGE_LIMIT`, evaluated against the inherited global params plus the UK-specific `vat_receipt_required` param.

---

## 20. Test Case Format *(v1.1)*

A BDL document may include an optional `tests` array. Each entry is a self-contained test case with an input and an expected verdict. Test cases serve as executable documentation and enable policy authors to verify behaviour at compile time and in CI pipelines.

### 20.1 TestCase Grammar

```
TestCase ::= {
  id: string,                              // unique within the document
  description?: string,
  params?: object,                         // param values to supply (if policy uses params)
  case: object,                            // the Case payload
  profile?: ExecutionProfile,              // if omitted, FULL_ENFORCEMENT is assumed
  expected: {
    verdict: Verdict,
    reason_codes?: string[],               // if present, all listed codes must appear in result
    required_fields?: string[]             // if present, all listed fields must appear as missing
  }
}
```

- `id` must be unique within the `tests` array.
- `expected.reason_codes` is an inclusive check: the result may contain additional reason codes.
- `expected.required_fields` is an inclusive check: the result may report additional missing fields.
- Test cases are **not evaluated at runtime** when the document is used for decisioning. They are evaluated only by a test runner or compiler.

### 20.2 Document Integration

Tests are declared at the top level of the BDL document under the `tests` key:

```
BDL_Document ::= {
  ...
  tests?: TestCase[]
}
```

### 20.3 Test Runner Semantics

A conformant test runner must:

1. For each TestCase, execute `evaluate_case` with the supplied `case`, `params`, and `profile`.
2. Compare the returned verdict against `expected.verdict`. A mismatch is a **test failure**.
3. If `expected.reason_codes` is present, verify all listed codes appear in the result. Missing codes are a **test failure**.
4. If `expected.required_fields` is present, verify all listed fields appear in `required_fields` of the result. Missing entries are a **test failure**.
5. Report a summary: total tests, passed, failed, and for each failure the TestCase `id`, expected values, and actual values.

### 20.4 Example

```yaml
tests:
  - id: TEST_MEAL_COMPLIANT
    description: "Meal over limit with receipt — should be compliant"
    case:
      expense:
        category: MEAL
        amount: 60
      evidence: [ITEMIZED_RECEIPT]
    expected:
      verdict: compliant
      reason_codes: [RECEIPT_MEETS_REQUIREMENT]

  - id: TEST_MEAL_MISSING_RECEIPT
    description: "Meal over limit without receipt — should need review"
    case:
      expense:
        category: MEAL
        amount: 60
      evidence: []
    expected:
      verdict: needs_review
      reason_codes: [ITEMIZATION_REQUIRED]

  - id: TEST_ADVANCE_BOOKING_VIOLATION
    description: "Domestic flight booked 7 days ahead — should violate 14-day rule"
    case:
      travel:
        air_scope: DOMESTIC
        advance_booking_days: 7
    expected:
      verdict: needs_review
      reason_codes: [DOMESTIC_BOOK_14_DAYS_ADVANCE]

  - id: TEST_PARAM_CUTOFF
    description: "Expense submitted after parameterised cutoff — should flag"
    params:
      submission_cutoff: "2025-03-31"
    case:
      expense:
        submitted_date: "2025-04-05"
        amount: 120
    expected:
      verdict: needs_review
      reason_codes: [SUBMISSION_AFTER_CUTOFF]
```

---

## 21. MCP Integration (Normative Interface)

The following tools define a minimal integration surface for deterministic BDL evaluation. v1.1 adds `params` to `evaluate_case` and `list_tests` / `run_tests` for test runner integration.

### 21.1 evaluate_case

**Input:**

- `case`: object — the Case payload
- `params?`: object — runtime param values *(v1.1)*
- `policy_id?`: string
- `version?`: string
- `profile?`: ExecutionProfile

**Output (DecisionResult):**

- `verdict`: Verdict
- `reason_codes`: string[]
- `required_fields?`: string[]
- `trace_id`: string
- `trace?`: object

### 21.2 get_schema

*(Unchanged from v1.0. Schema output should now include declared params.)*

### 21.3 list_policies

*(Unchanged from v1.0.)*

### 21.4 get_trace

*(Unchanged from v1.0. Traces now include resolved param values and composition origin per §9.4.)*

### 21.5 list_tests *(v1.1)*

**Input:**

- `policy_id?`: string
- `version?`: string

**Output:** Array of TestCase summaries (id, description, expected verdict) from the document's `tests` array.

### 21.6 run_tests *(v1.1)*

**Input:**

- `policy_id?`: string
- `version?`: string
- `test_ids?`: string[] — if omitted, all tests are run

**Output:**

- `total`: number
- `passed`: number
- `failed`: number
- `results`: array of `{ id, passed, expected, actual }` for each test

---

## 22. Meta Grammar (Compiler-Only)

```
Meta ::= {
  compiler_confidence?: number,   // 0.0 – 1.0
  assumptions?: string[]
}
```

Runtime must ignore `meta`; it is for compiler or tooling use only.

---

*End of Technical Specification v1.1*
