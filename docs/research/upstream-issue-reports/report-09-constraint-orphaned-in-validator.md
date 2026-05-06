# Report 09 — `DatastreamDataComponent.Constraint` orphaned in validator

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-09-constraint-orphaned-in-validator.md`](../upstream-issues/plan-09-constraint-orphaned-in-validator.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#12** (`DatastreamDataComponent.Constraint` orphaned in validator) |
| Source fork issue | `OS4CSAPI/connected-systems-go#21` (closed by upstream addition of `matchesNilValue` helper); this filing is the `Constraint` sibling explicitly scoped out by the eval. |
| Research plan | [`../upstream-issues/plan-09-constraint-orphaned-in-validator.md`](../upstream-issues/plan-09-constraint-orphaned-in-validator.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P2** — spec conformance / observation correctness; out-of-constraint values silently accepted |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Parent #21 fix in place; `Constraint`
remains orphaned — model-declared, GET-round-tripped, validator
never consults it.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates
```

**Parent #21 fix (helper present):**

```go
// internal/api/observation_schema_validation.go
func matchesNilValue(component *domains.DatastreamDataComponent, value any) bool {
    if component == nil || len(component.NilValues) == 0 {
        return false
    }
    for _, nv := range component.NilValues {
        var decoded any
        if err := json.Unmarshal(nv.Value, &decoded); err != nil {
        ...
}
```

Called early in `validateDataComponentValue`:

```go
if matchesNilValue(component, value) {
    return nil
}
```

**Validator's leaf-scalar branches** (post-#21):

```go
case "boolean":
    if _, ok := value.(bool); !ok { ... }
case "count":
    if !isIntegerNumber(value) { ... }
case "quantity":
    if !isNumber(value) { ... }
case "time", "category", "text":
    if _, ok := value.(string); !ok { ... }
```

Six leaf-scalar branches across four `case` arms. Each branch
performs a type-shape check; **none consult
`component.Constraint`.**

**`Constraint` is absent from the entire validator package:**

```text
$ git grep -nE 'Constraint' upstream/main -- internal/api/
(no output — zero matches)
```

**`Constraint` IS declared at the model layer and round-trips on GET:**

```text
$ git grep -nE 'Constraint' upstream/main -- internal/model/domains/datastream.go
internal/model/domains/datastream.go:233:    Constraint *DatastreamConstraint `json:"constraint,omitempty"`
internal/model/domains/datastream.go:315:// DatastreamConstraint maps common SWE constraint shapes.
internal/model/domains/datastream.go:316:type DatastreamConstraint struct {
```

The `DatastreamConstraint` shape (fork-specific simplification of
SWE Common 3.0 §"AllowedValues"/"AllowedTokens"/"AllowedTimes"
constraint families):

```go
type DatastreamConstraint struct {
    Type               string          `json:"type,omitempty"`
    Values             json.RawMessage `json:"values,omitempty"`
    Intervals          json.RawMessage `json:"intervals,omitempty"`
    Pattern            string          `json:"pattern,omitempty"`
    SignificantFigures *int            `json:"significantFigures,omitempty"`
    Extensions         common_shared.Properties `json:"extensions,omitempty"`
}
```

Five normative fields cover the common SWE Common shapes:
`Values` (discrete allowed set), `Intervals` (numeric ranges),
`Pattern` (string regex), `SignificantFigures` (numeric
precision), `Type` (constraint kind discriminator).

**Asymmetry confirmed.** The same model-layer-vs-validator gap that
parent #21 closed for `NilValues` persists for `Constraint`.

**Live re-verification:** not captured this round. POST of a
constrained Datastream + Observations requires authenticated admin
endpoints not exposed on `csapi-go-upstream`. The defect is
structural — `Constraint` is unreferenced in the validator package
— and parent #21's matrix transitively applies (same shape).

## 2. Static evidence

Source: [`../evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md)
"Adjacent finding (not in original body) — sibling fields are
similarly orphaned" §, refreshed in §1 above.

The defect shape is the **identical pattern** parent #21 fixed:

| Layer | `NilValues` (parent fix) | `Constraint` (this filing) |
|---|---|---|
| Model declaration | `domains.DatastreamDataComponent.NilValues` | `domains.DatastreamDataComponent.Constraint` |
| JSON tag | `json:"nilValues,omitempty"` | `json:"constraint,omitempty"` |
| GET round-trip | ✓ | ✓ |
| Validator consultation | ✓ (post-#21: `matchesNilValue` helper) | ✗ (zero references in `internal/api/`) |

The model-layer plumbing is correct. A client can declare a
constraint on a Datastream's DataComponent, and that constraint is
persisted and returned on GET. But on observation POST, the
validator never reads `component.Constraint` and never enforces
the declared bounds — an out-of-constraint value passes the type
check, then commits to the observations table as if validly
in-bounds.

## 3. Live evidence

Not captured this round (auth-gated POST). Justification in §1.
The 4-row matrix sketched in plan §6 (declare a constrained
Datastream, POST in-bounds → 201 control, POST out-of-bounds →
**currently 201, defect**, GET round-trip preserves the constraint)
follows deterministically from the static evidence: the validator
package does not reference `Constraint` at any branch, so no
out-of-bounds rejection is possible.

Parent #21's pre-fix matrix in
[`../evidence/issue-021/live-test-2026-04-30.md`](../evidence/issue-021/live-test-2026-04-30.md)
demonstrates the same model-vs-validator asymmetry on the same
file; the structural defect for `Constraint` is the symmetric case.

## 4. Spec authority

Direct binding via OGC 23-002 (CSAPI Part 2 schemas) + OGC 23-011r1
(SWE Common 3.0).

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 Datastream / DataComponent schemas | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms `constraint` is a normative property on every scalar `DataComponent` subclass per the bundled schemas. **Primary citation.** |
| **OGC 23-011r1** — OGC SWE Common 3.0 | "OGC SWE Common 3.0 (OGC 23-011r1)" under *OGC Standards* — verified canonical entry | Authority for the constraint shape semantics: discrete-`Values` enumeration, `Intervals` range, `Pattern` regex, `SignificantFigures` precision. The fork's `DatastreamConstraint` struct simplifies these but maps directly to SWE Common 3.0 constraint families. |
| **OGC 23-001** — CSAPI Part 1 / Observation conformance class | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms observation submission must conform to the datastream's declared schema; constraint enforcement is part of that conformance contract. |
| **RFC 7807** §3 (problem-detail "identify the problem") | "RFC 7807 — Problem Details for HTTP APIs" under *IETF RFCs* | Supporting: when constraint enforcement IS added, the rejection message should be specific (e.g., `"value <V> outside declared interval [<min>, <max>]"`), not generic. |

**Out-of-scope sources (not cited):** OAS 3.0.3, JSON Schema 2020-12, RFC 7493.

> **References-list verification.** Plan §7 speculated SWE Common as
> "OGC 12-000". The current canonical entry is **OGC 23-011r1**
> (SWE Common 3.0, 2024). This report cites the canonical entry.
> No reference-list gap.

## 5. Alternatives considered (internal-only)

Only one viable shape — mirror parent #21's `matchesNilValue`
helper-function pattern, dispatching by `DatastreamConstraint.Type`
and the leaf-scalar component type.

### Option A — Add `matchesConstraint` helper, call at each leaf-scalar branch (recommended)

Pseudocode:

```go
// validates a value against component.Constraint; returns nil if no constraint
// declared or value satisfies it; returns *decodeError otherwise.
func validateAgainstConstraint(component *domains.DatastreamDataComponent, value any) error {
    if component == nil || component.Constraint == nil {
        return nil
    }
    c := component.Constraint
    switch component.Type {
    case "quantity", "count":
        // dispatch on c.Type / c.Intervals / c.Values for numeric semantics
    case "time":
        // dispatch on c.Intervals / c.Values for temporal semantics
    case "category", "text":
        // dispatch on c.Values (allowed-tokens) / c.Pattern (regex)
    case "boolean":
        // no constraint semantics defined; return nil
    }
    ...
}
```

Inserted at each of the six leaf-scalar branches after the type
check passes:

```go
case "quantity":
    if !isNumber(value) { return ...wrong-type... }
    if err := validateAgainstConstraint(component, value); err != nil {
        return err
    }
```

- Mirrors parent #21's helper-then-branch-call shape exactly.
- Risk: the **comparison semantics** are non-trivial (numeric
  intervals vs. discrete-values vs. regex pattern vs. allowed
  tokens). Per plan §8 Q1, this report defers detailed
  comparison-semantics design to the fix PR. The issue body
  articulates the *shape* (one helper, called at each leaf-scalar
  branch, dispatching by component type and constraint kind), not
  line-level implementation.

**Lead with Option A only.** No alternative shapes considered.

## 6. Recommended fix

**Option A.** Mirror parent #21's pattern:

1. Add `validateAgainstConstraint(component, value)` helper next to
   `matchesNilValue` in
   `internal/api/observation_schema_validation.go`.
2. Call the helper at each of the six leaf-scalar branches
   (`boolean`, `count`, `quantity`, `time`, `category`, `text`)
   after the existing type check passes.
3. Comparison semantics dispatch by component-type tag and
   `DatastreamConstraint` field presence:
   - **numeric** (`quantity`, `count`): consult `Intervals` for
     range checks, `Values` for discrete-allowed lists,
     `SignificantFigures` for precision (advisory or normative — fix
     PR decides).
   - **temporal** (`time`): consult `Intervals` for range,
     `Values` for discrete-allowed instants.
   - **textual** (`category`, `text`): consult `Values` for
     allowed-tokens, `Pattern` for regex match.
   - **boolean**: no SWE Common constraint semantics defined —
     return nil; constraint is inert.
4. On rejection, return a `*decodeError` with a specific message
   identifying the violated constraint kind (per RFC 7807 §3).

Implementation surface: one new helper plus one helper-call line at
each of six leaf-scalar branches, all within
`observation_schema_validation.go`. No new types, no model
changes, no schema migration.

## 7. Scope guard

What NOT to touch as part of this filing:

- `matchesNilValue` and the `NilValues` consultation — parent #21
  fix in place, working correctly.
- Model layer — `DatastreamDataComponent.Constraint` and
  `DatastreamConstraint` are correctly declared and round-trip on
  GET; no model changes needed.
- `Updatable` enforcement — separate filing, plan-10 / report-10.
  Different concern (PUT/PATCH path semantics, not POST validation).
- Schema migration — `constraint` JSON shape is already persisted
  and returned by GET; no DB changes.
- The 6-branch leaf-scalar switch structure itself — additive
  changes only; do not reorganize.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#21` (parent) was the umbrella P2
  finding for "model declares it, GET round-trips it, validator
  ignores it". Closed by the addition of `matchesNilValue` and its
  call in `validateDataComponentValue`. The eval flagged
  `Constraint` and `Updatable` as adjacent findings of the
  identical shape and explicitly recommended separate filings.
- This filing addresses `Constraint`. Plan-10 / report-10
  addresses `Updatable`.
- No fork-side patch; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Comparison semantics design | **Defer line-level design to fix PR.** Issue body articulates the helper-shape and per-component-type dispatch table; does not over-prescribe range-vs-list-vs-pattern implementation. |
| Apply uniformly to all leaf-scalar branches? | **Yes.** §1 confirms 6 branches in the validator (plan said 7; actual count is 6 across 4 case arms). Helper called at each. `boolean` branch returns inert (no SWE Common constraint semantics). |
| Bundle `Updatable` (plan-10)? | **No.** Eval recommended separate filings; `Updatable` is a different code path (PUT/PATCH, not POST validation). One-line "see related" footer when plan-10 is filed. |
| Severity P2 vs P3? | **P2 retained.** Spec conformance defect; clients declaring constraints get no enforcement; out-of-constraint values silently committed. Direct functional impact on observation correctness. |
| Live reproducer required? | **No.** Auth-gated; defect is structural (zero `Constraint` references in validator package); static evidence + parent #21's matrix as transitive precedent. |
| Cite eval/evidence paths? | **Yes**, validation-chain footer. |
| References-list canonical entry for SWE Common | **OGC 23-011r1** (SWE Common 3.0). Plan §7 speculated "OGC 12-000"; that's outdated. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P2] DatastreamDataComponent.Constraint orphaned in validator — sibling of issue #21 (NilValues)`

**Labels:** `bug`, `spec-conformance`

---

### Context

Issue #21 (closed) added the `matchesNilValue` helper to
`internal/api/observation_schema_validation.go` and called it at
the start of `validateDataComponentValue`, closing the gap where
`DatastreamDataComponent.NilValues` was model-declared and
GET-round-tripped but never consulted by the validator.

The same gap persists for the sibling field `Constraint` on the
same model type. `git grep` shows zero references to `Constraint`
in the entire `internal/api/` package, while the field is declared
on `DatastreamDataComponent` at the model layer
(`internal/model/domains/datastream.go:233`) and round-trips on
GET via the `DatastreamConstraint` struct (line 316).

Verified on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

A client declaring a `constraint` on a Datastream's DataComponent
(numeric `intervals`/`values`, string `pattern`/`values`, etc.)
receives no enforcement on observation POST. Out-of-constraint
values pass the type-shape check and commit to the observations
table as if in-bounds.

### Static evidence

`internal/api/observation_schema_validation.go` (post-#21):

```go
// matchesNilValue reports whether value equals any declared nilValue in component.
func matchesNilValue(component *domains.DatastreamDataComponent, value any) bool {
    if component == nil || len(component.NilValues) == 0 {
        return false
    }
    for _, nv := range component.NilValues {
        var decoded any
        if err := json.Unmarshal(nv.Value, &decoded); err != nil { ...
}

// In validateDataComponentValue:
if matchesNilValue(component, value) {
    return nil
}

switch component.Type {
case "boolean":
    if _, ok := value.(bool); !ok { ... }
case "count":
    if !isIntegerNumber(value) { ... }
case "quantity":
    if !isNumber(value) { ... }
case "time", "category", "text":
    if _, ok := value.(string); !ok { ... }
}
```

`Constraint` reference audit:

```text
$ git grep -nE 'Constraint' upstream/main -- internal/api/
(zero matches)

$ git grep -nE 'Constraint' upstream/main -- internal/model/domains/datastream.go
internal/model/domains/datastream.go:233:    Constraint *DatastreamConstraint `json:"constraint,omitempty"`
internal/model/domains/datastream.go:315:// DatastreamConstraint maps common SWE constraint shapes.
internal/model/domains/datastream.go:316:type DatastreamConstraint struct { ...
```

The `DatastreamConstraint` struct (declared, JSON-tagged,
GET-round-tripped):

```go
type DatastreamConstraint struct {
    Type               string          `json:"type,omitempty"`
    Values             json.RawMessage `json:"values,omitempty"`
    Intervals          json.RawMessage `json:"intervals,omitempty"`
    Pattern            string          `json:"pattern,omitempty"`
    SignificantFigures *int            `json:"significantFigures,omitempty"`
    Extensions         common_shared.Properties `json:"extensions,omitempty"`
}
```

### Asymmetry table

| Aspect | `NilValues` (post-#21) | `Constraint` (this filing) |
|---|---|---|
| Model declaration | ✓ | ✓ |
| JSON tag | ✓ | ✓ |
| GET round-trip | ✓ | ✓ |
| Validator consultation | ✓ (`matchesNilValue` helper) | ✗ (zero references in `internal/api/`) |

### Live evidence

Not included in this filing — POST of a constrained Datastream and
its Observations requires authenticated admin endpoints. The
defect is structural (zero `Constraint` references in the
validator package); the behavior follows deterministically from
the static evidence. Parent issue #21's pre-fix matrix demonstrates
the same model-vs-validator asymmetry pattern on the same file.

### Recommended fix

Mirror the parent-#21 pattern: add a `validateAgainstConstraint`
helper next to `matchesNilValue`, call it at each of the six
leaf-scalar branches after the existing type check passes:

```go
func validateAgainstConstraint(component *domains.DatastreamDataComponent, value any) error {
    if component == nil || component.Constraint == nil {
        return nil
    }
    c := component.Constraint
    switch component.Type {
    case "quantity", "count":
        // dispatch on c.Intervals (numeric range), c.Values (discrete set), c.SignificantFigures (precision)
    case "time":
        // dispatch on c.Intervals (temporal range), c.Values (discrete instants)
    case "category", "text":
        // dispatch on c.Values (allowed tokens), c.Pattern (regex match)
    case "boolean":
        return nil  // no SWE Common constraint semantics
    }
    ...
}
```

Comparison semantics dispatch by component-type tag and
`DatastreamConstraint` field presence; line-level implementation
deferred to the fix PR. Rejection should return a `*decodeError`
with a specific message identifying the violated constraint kind
(per RFC 7807 §3).

Implementation surface: one new helper + one helper-call line at
each of six leaf-scalar branches, all within
`observation_schema_validation.go`. No new types, no model
changes, no schema migration.

### Spec authority

- **OGC 23-002** CSAPI Part 2 — Datastream / DataComponent schemas
  declare `constraint` as a normative property on scalar
  DataComponent subclasses. **Primary citation.**
- **OGC 23-011r1** SWE Common 3.0 — defines the constraint shape
  semantics (discrete `Values` enumeration, `Intervals` range,
  `Pattern` regex, `SignificantFigures` precision). The fork's
  `DatastreamConstraint` struct simplifies but maps directly to
  these families.
- **OGC 23-001** CSAPI Part 1 — observation submission must
  conform to the datastream's declared schema; constraint
  enforcement is part of that conformance contract.
- **RFC 7807** §3 — when enforcement is added, rejection messages
  should identify the specific constraint kind violated.

### Severity

**P2** — spec conformance / observation correctness. Clients
declaring constraints get no enforcement; out-of-constraint values
commit silently, corrupting the observation record's adherence to
the declared datastream schema. Direct functional impact.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-021.md`](../issue-evaluations/issue-021.md) §"Adjacent finding"
- Evidence (static): [`docs/research/evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md)
- Evidence (live, parent matrix): [`docs/research/evidence/issue-021/live-test-2026-04-30.md`](../evidence/issue-021/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-021/spec-authority-2026-04-30.md`](../evidence/issue-021/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-09-constraint-orphaned-in-validator.md`](../upstream-issues/plan-09-constraint-orphaned-in-validator.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #12

**See also:** symmetric filing for `Updatable` is in preparation
(plan-10 / report-10).

## 11. Filing record

- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/8>
- **Filed:** 2026-05-05
- **Filed by:** orchestrator (via mcp_io_github_git_issue_write)
- **Audit verdict:** `pass` (after 2 in-place broken-link fixes) — see [report-audit-log.md#report-09-constraint-orphaned-in-validator](../report-audit-log.md#report-09-constraint-orphaned-in-validator)

