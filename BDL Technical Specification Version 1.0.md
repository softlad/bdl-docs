# Business Decision Language (BDL) — Technical Specification

**Version:** 1.0  
**Status:** Authoritative reference for BDL document grammar, execution, and integration.

---

## 1. Conceptual Model

### 1.1 Case

A **Case** is an immutable JSON object supplied at runtime. It represents a single decision instance. Field paths use dot notation (e.g. `expense.amount`, `request.item`). The shape of a Case is defined by the policy’s generated schema; no fixed global schema is required.

### 1.2 BDL Document (Policy)

A **BDL document** is a single YAML or JSON artifact that contains:

- Document-level metadata (policy_id, version, effective dates, jurisdiction, defaults)
- Optional **tables** — lookup tables for lookup-based decisioning (see §2, TableDefinition)
- An ordered list of **statements**

A **Policy Package** is a versioned bundle that may include one or more BDL documents, a generated input schema, lookup tables, and metadata. **Lookup-based decisioning** is when an output (e.g. price, rate, tier) is determined by matching the Case to a table row via a composite key; the Value grammar’s **Lookup** and DEFINE rules use these tables.

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
  ir_version: string,           // e.g. "1.0"
  policy_id: string,
  policy_name?: string,
  version: string,
  effective: { start: date, end?: date },
  jurisdiction?: string[],
  priority_model: "explicit",
  defaults: Defaults,
  tables?: TableDefinition[],
  statements: Statement[]
}
```

- **defaults**: e.g. `{ on_missing: needs_info | needs_review, on_error: needs_review }`
- **tables**: optional array of lookup tables. Each table has an id (referenced by Lookup.table), key columns, a value column, and rows. Used for lookup-based decisioning (pricing, rate cards, eligibility). See TableDefinition below.
- Only the fields listed are permitted at the top level.

### 2.1 TableDefinition (lookup tables)

```
TableDefinition ::= {
  id: string,                   // unique; referenced by Lookup.table
  key_columns: string[],         // column names forming composite key (order matches Lookup.key)
  value_column: string,         // column name for the output value
  rows: Row[]
}

Row ::= object                  // keys = key_columns ∪ { value_column }; values = literals
```

At runtime, a **Lookup** value supplies `table` (id) and `key` (array of FieldRefs). The engine reads the Case (and DEFINE-derived context) at each key path, finds the row whose key_columns match those values in order, and returns that row’s value_column. No match → implementation may treat as missing or apply on_error/defaults.

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

FieldRef ::= string   // dot-notation path, e.g. "expense.amount"
```

- **eq / neq**: equality (value may be string, number, boolean, null).
- **lt / lte / gt / gte**: numeric comparison; the value in the Case at FieldRef is compared to the given Value.
- **in**: value at FieldRef must be in the given array.
- **exists**: value at FieldRef is not undefined and not null.
- **contains**: for arrays, array includes Value; for strings, string contains Value (string).

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

- **Lookup**: used for **lookup-based decisioning**. `table` names a table in the document’s `tables` (see §2.1). `key` is an array of FieldRefs; the engine reads the Case (and DEFINE context) at each path, in order, to form the composite key. The row in that table whose key_columns equal those values is selected; the result is that row’s value_column. Commonly used in DEFINE to set an output (e.g. output.unit_price) from a table.
- **Arithmetic**: must be side-effect free and finite.

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

- **ALLOW**: when applies_when is true and the value at `field` is in `values`, the statement’s on_apply outcome is used.
- **FORBID**: when applies_when is true and the value at `field` is in `values`, the statement’s on_violation outcome is used.

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

- **on_apply**: when the statement’s condition and rule are satisfied (e.g. ALLOW matches, REQUIRE satisfied).
- **on_violation**: when the rule is violated (e.g. FORBID matches, LIMIT exceeded).
- **on_missing**: when required fields or evidence are missing.
- **on_error**: when evaluation of the statement throws or fails.

**override**: when true, this outcome can override a lower-priority statement’s outcome.  
**halt**: when true, evaluation may stop (implementation-defined).

### 7.1 Verdicts

A **verdict** is the outcome classification for a case. It indicates whether the case is permitted, prohibited, or requires further action. Each verdict has a defined meaning:

| Verdict | Meaning | Use |
|--------|--------|-----|
| **compliant** | The case satisfies the rule. The statement’s condition and rule were met (e.g. ALLOW matched, REQUIRE satisfied, LIMIT not exceeded). The case may proceed as evaluated. | Used in **on_apply** when the rule passes. |
| **non_compliant** | The case violates the rule. The rule was not satisfied (e.g. FORBID matched, REQUIRE failed, LIMIT exceeded). The case should be treated as not permitted by this statement. | Used in **on_violation** when the rule fails. |
| **needs_info** | Required data or evidence is missing. The engine cannot determine compliance or violation without additional case data. The case should not be approved or denied solely on this statement until the missing information is supplied. | Used in **on_missing** (or document default) when fields or evidence are absent. |
| **needs_review** | The case requires human or downstream review. Used when policy explicitly routes for approval, when evaluation encountered an error, or when missing data is handled by sending the case to a reviewer rather than blocking on info. | Used in **on_apply** (e.g. ROUTE to approval), **on_missing**, or **on_error** (or document default). |
| **no_change** | No effect on the overall outcome. The statement adds context (e.g. TAG labels, audit metadata) but does not by itself set a compliant/non_compliant/needs_info/needs_review result. The effective verdict comes from other statements or remains unchanged. | Used when the statement only tags or annotates (e.g. TAG with **on_apply**). |

- **compliant** and **non_compliant** are definitive for the statement: the rule was evaluated and passed or failed.
- **needs_info** and **needs_review** are first-class; they do not imply a value for missing data and do not auto-approve or auto-deny.
- The **final verdict** for a case is determined by evaluation order, priority, and override rules (see §9.2); it is one of the five verdicts above.

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

Citations link statements to source policy documents for audit and explanation.

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

---

## 10. Execution Profiles

Execution scope is controlled by a **profile** supplied at runtime with the Case. The profile is **not** part of the BDL document.

### 10.1 Execution Request

```
ExecutionRequest ::= {
  profile?: ExecutionProfile,
  case: object,
  policy_id?: string,
  version?: string
}
```

If `profile` is omitted, the runtime behaves as for full enforcement (all statement types, enforce missing data).

### 10.2 ExecutionProfile

```
ExecutionProfile ::= {
  evaluate_types: StatementType[],
  missing_data_behavior?: "enforce" | "ask" | "ignore"
}
```

- **evaluate_types**: allowlist of statement types. Only statements whose `type` is in this list are evaluated.
- **missing_data_behavior**:
  - **enforce**: use statement outcomes and document defaults for missing data (needs_info / needs_review).
  - **ask**: prefer needs_info so the caller can gather data.
  - **ignore**: do not block on missing data; skip statements that cannot be evaluated and record in trace.

The effective profile must be recorded in the trace.

### 10.3 Reference Profiles (Non-Normative)

| Profile name                 | evaluate_types                                                    | missing_data_behavior |
|-----------------------------|-------------------------------------------------------------------|------------------------|
| ADVISORY_PERMISSIBILITY     | DEFINE, ALLOW, FORBID, TAG                                       | ignore                 |
| CONSTRAINT_CHECK            | DEFINE, ALLOW, FORBID, LIMIT, TAG                                | ask                    |
| FULL_ENFORCEMENT            | DEFINE, ALLOW, FORBID, LIMIT, REQUIRE, ROUTE, TAG                | enforce                |

---

## 11. Hard Constraints (Compiler / Authoring)

When generating or authoring BDL:

1. Use only constructs defined in this specification.
2. Do not introduce new statement types, comparison operators, or verdicts.
3. Every statement should include at least one citation.
4. Represent ambiguity via needs_info or needs_review outcomes, not by omitting outcomes.
5. Do not encode business meaning outside statements (e.g. in comments only).
6. All behavior must be explainable via the trace.

---

## 12. Example: Casual Friday (FORBID + ALLOW)

**Policy:** Jeans are not allowed by default; they are allowed on Fridays if there is no client-facing meeting.

**BDL:**

```yaml
- id: DRESS_FORBID_JEANS_DEFAULT
  type: FORBID
  priority: 50
  applies_when:
    eq: [request.item, JEANS]
  rule:
    field: request.item
    values: [JEANS]
  outcomes:
    on_violation:
      verdict: non_compliant
      reason_code: JEANS_NOT_ALLOWED

- id: DRESS_ALLOW_JEANS_FRIDAY
  type: ALLOW
  priority: 90
  applies_when:
    all:
      - eq: [request.item, JEANS]
      - eq: [context.day_of_week, FRIDAY]
      - neq: [context.is_client_meeting, true]
  rule:
    field: request.item
    values: [JEANS]
  outcomes:
    on_apply:
      verdict: compliant
      reason_code: CASUAL_FRIDAY
      override: true
```

**Sample Case (compliant):**

```json
{
  "request": { "item": "JEANS" },
  "context": { "day_of_week": "FRIDAY", "is_client_meeting": false }
}
```

**Sample Case (non-compliant):**

```json
{
  "request": { "item": "JEANS" },
  "context": { "day_of_week": "MONDAY" }
}
```

---

## 13. Example: Expense Meal (REQUIRE)

**Policy:** Meals over $25 require an itemized receipt.

**BDL:**

```yaml
- id: MEAL_REQUIRE_ITEMIZATION
  type: REQUIRE
  priority: 80
  applies_when:
    all:
      - eq: [expense.category, MEAL]
      - gt: [expense.amount, 25]
  rule:
    require_evidence: [ITEMIZED_RECEIPT]
  outcomes:
    on_apply:
      verdict: compliant
      reason_code: RECEIPT_MEETS_REQUIREMENT
    on_missing:
      verdict: needs_review
      reason_code: ITEMIZATION_REQUIRED
```

**Sample Case (compliant):**

```json
{
  "expense": { "category": "MEAL", "amount": 60 },
  "evidence": ["ITEMIZED_RECEIPT"]
}
```

**Sample Case (needs_review):**

```json
{
  "expense": { "category": "MEAL", "amount": 60 },
  "evidence": []
}
```

---

## 14. Example: Approval Routing (ROUTE)

**Policy:** Purchases over $10,000 require VP approval.

**BDL:**

```yaml
- id: PURCHASE_ROUTE_VP_APPROVAL
  type: ROUTE
  priority: 100
  applies_when:
    gt: [purchase.amount, 10000]
  rule:
    to: VP_APPROVAL
  outcomes:
    on_apply:
      verdict: needs_review
      reason_code: VP_APPROVAL_REQUIRED
```

**Sample Case:**

```json
{
  "purchase": { "amount": 15000 }
}
```

Result: verdict needs_review, route to VP_APPROVAL.

---

## 15. Example: Threshold (LIMIT)

**Policy:** Domestic flights must be booked at least 14 days in advance.

**BDL:**

```yaml
- id: DOMESTIC_ADVANCE_BOOKING
  type: LIMIT
  priority: 65
  applies_when:
    eq: [travel.air_scope, DOMESTIC]
  rule:
    field: travel.advance_booking_days
    op: gte
    value: 14
  outcomes:
    on_violation:
      verdict: needs_review
      reason_code: DOMESTIC_BOOK_14_DAYS_ADVANCE
```

**Sample Case (compliant):**

```json
{
  "travel": { "air_scope": "DOMESTIC", "advance_booking_days": 21 }
}
```

**Sample Case (violation):**

```json
{
  "travel": { "air_scope": "DOMESTIC", "advance_booking_days": 7 }
}
```

---

## 16. MCP Integration (Normative Interface)

The following tools define a minimal integration surface for deterministic BDL evaluation. Request/response shapes are normative; transport (e.g. MCP) is implementation-defined.

### 16.1 evaluate_case

**Input:**

- `case`: object — the Case payload
- `policy_id?`: string
- `version?`: string
- `profile?`: ExecutionProfile

**Output (DecisionResult):**

- `verdict`: Verdict
- `reason_codes`: string[]
- `required_fields?`: string[] — fields or evidence still missing (when applicable)
- `trace_id`: string — reference to full trace
- `trace?`: object — optional inline trace (profile, fired statements, citations)

### 16.2 get_schema

**Input:**

- `policy_id?`: string
- `version?`: string

**Output:** JSON Schema (or equivalent) describing the expected Case shape and, where applicable, allowed values for enumerated fields.

### 16.3 list_policies

**Input:** none

**Output:** List of policy summaries (policy_id, version, effective dates, etc.).

### 16.4 get_trace

**Input:** `trace_id`: string

**Output:** Full trace object for the given evaluation (evaluated statements, values used, citations, effective profile).

---

## 17. Meta Grammar (Compiler-Only)

```
Meta ::= {
  compiler_confidence?: number,   // 0.0 – 1.0
  assumptions?: string[]
}
```

Runtime must ignore `meta`; it is for compiler or tooling use only.

---

*End of Technical Specification*
