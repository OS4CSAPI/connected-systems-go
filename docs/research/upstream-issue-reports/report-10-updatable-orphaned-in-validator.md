# Report 10 — `DatastreamDataComponent.Updatable` orphaned in validator

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-10-updatable-orphaned-in-validator.md`](../upstream-issues/plan-10-updatable-orphaned-in-validator.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#13** (`DatastreamDataComponent.Updatable` orphaned in validator) |
| Source fork issue | `OS4CSAPI/connected-systems-go#21` (closed by upstream addition of `matchesNilValue`); this filing is the `Updatable` sibling explicitly scoped out by the eval (paired with `Constraint`/report-09). |
| Research plan | [`../upstream-issues/plan-10-updatable-orphaned-in-validator.md`](../upstream-issues/plan-10-updatable-orphaned-in-validator.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P3** — narrower surface than `Constraint` (update path only); recoverable via subsequent GET. |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Defect confirmed: `Updatable` is declared
at the model layer but referenced **zero times** in
`internal/api/`. The two schema-update handlers do bare
decode→repo→204 with no diff or gating.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates
```

**`Updatable` reference audit:**

```text
$ git grep -nE 'Updatable' upstream/main -- internal/api/
(zero matches — same shape as parent #21 pre-fix)

$ git grep -nE 'Updatable' upstream/main -- internal/model/
internal/model/common_shared/characteristics.go:96:    Updatable *bool `json:"updateable,omitempty"`
internal/model/domains/datastream.go:223:              Updatable *bool `json:"updatable,omitempty"`
internal/model/generators/generators_common_shared.go:107:        Updatable: &up,
```

**Update-handler surface (enumerated per plan §6):**

```text
$ git grep -nE 'func.*Update(Datastream|DataStream|ControlStream)' upstream/main -- internal/api/
internal/api/control_stream_handler.go:246: func (h *ControlStreamHandler) UpdateControlStreamSchema(w http.ResponseWriter, r *http.Request)
internal/api/datastream_handler.go:159:     func (h *DatastreamHandler)     UpdateDatastream(w http.ResponseWriter, r *http.Request)
internal/api/datastream_handler.go:233:     func (h *DatastreamHandler)     UpdateDatastreamSchema(w http.ResponseWriter, r *http.Request)
```

Three update handlers in scope — two for schema (which contains
the DataComponents whose `Updatable` flag is at issue) and one for
the Datastream resource as a whole.

**`UpdateDatastreamSchema` body** (HEAD `df6da0d`,
`internal/api/datastream_handler.go:233`):

```go
func (h *DatastreamHandler) UpdateDatastreamSchema(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "dataStreamId")

    var schema domains.DatastreamSchema
    if err := render.DecodeJSON(r.Body, &schema); err != nil {
        render.Status(r, http.StatusBadRequest)
        render.JSON(w, r, map[string]string{"error": "Invalid request body"})
        return
    }

    if err := h.repo.UpdateSchema(id, &schema); err != nil {
        ...500...
    }

    w.WriteHeader(http.StatusNoContent)
}
```

**`UpdateControlStreamSchema` body** (HEAD `df6da0d`,
`internal/api/control_stream_handler.go:246`):

```go
func (h *ControlStreamHandler) UpdateControlStreamSchema(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "controlStreamId")

    var schema domains.ControlStreamSchema
    if err := render.DecodeJSON(r.Body, &schema); err != nil {
        ...400...
    }

    if err := h.repo.UpdateSchema(id, &schema); err != nil {
        ...500...
    }

    w.WriteHeader(http.StatusNoContent)
}
```

Both schema-update handlers are **bare decode→repo→204**:

- No fetch of the existing schema.
- No diff against the new schema to identify which components are
  being changed.
- No consultation of `component.Updatable` on any component.
- No rejection on edits to `updatable=false` components.

The body of `UpdateDatastream` (line 159) is the resource-level
PUT and is similarly silent on `Updatable`.

**Sanity (parent #21 fix in place):**

```text
$ git show upstream/main:internal/api/observation_schema_validation.go |
    Select-String 'NilValues' -Context 1,1
> if component == nil || len(component.NilValues) == 0 {
>     return false
> for _, nv := range component.NilValues {
```

**Live re-verification:** not captured this round. The required
sequence (POST datastream w/ `updatable=false` component → GET to
verify round-trip → PATCH/PUT → GET to confirm silent acceptance)
requires authenticated admin endpoints not exposed on
`csapi-go-upstream`. Defect is structural (zero `Updatable`
references in `internal/api/` plus bare-handler bodies above) and
parent #21's matrix transitively applies (same shape, same audit).

## 2. Static evidence

Source: [`../evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md)
"Adjacent finding (not in original body)" §, refreshed in §1
above.

The defect shape is the **third instance** of the audit pattern
parent #21 surfaced:

| Layer | `NilValues` (parent #21, fixed) | `Constraint` (report-09) | `Updatable` (this filing) |
|---|---|---|---|
| Model declaration | ✓ | ✓ | ✓ |
| JSON tag | ✓ | ✓ | ✓ |
| GET round-trip | ✓ | ✓ | ✓ |
| Validator/handler consultation | ✓ (post-#21) | ✗ | ✗ |
| Affected handler | POST validator | POST validator | PUT/PATCH update handlers |

**Adjacent finding — JSON-tag inconsistency:**

```text
internal/model/common_shared/characteristics.go:96:  Updatable *bool `json:"updateable,omitempty"`   ← "updateable" (extra e)
internal/model/domains/datastream.go:223:           Updatable *bool `json:"updatable,omitempty"`    ← canonical SWE Common spelling
```

Two model declarations of the same conceptual flag use **different
JSON tag spellings** (`updateable` vs `updatable`). This is a
separate defect (likely client-incompatible across the two
resource families) and is **out of scope for this filing**. Noted
here for the maintainer's awareness; recommend a separate followup
if the maintainer wants it tracked.

## 3. Live evidence

Not captured this round (auth-gated PUT/PATCH; multi-step
sequence). Justification in §1. The 4-step matrix sketched in
plan §6 follows deterministically from the static evidence: with
zero `Updatable` references in the validator package and the
schema-update handlers performing a bare
`DecodeJSON → UpdateSchema → 204`, no rejection path can exist for
edits to `updatable=false` components.

Parent #21's pre-fix matrix in
[`../evidence/issue-021/live-test-2026-04-30.md`](../evidence/issue-021/live-test-2026-04-30.md)
demonstrates the same model-vs-handler asymmetry pattern; this
filing is the third occurrence of that pattern.

## 4. Spec authority

Direct binding via OGC 23-002 + OGC 23-011r1 (SWE Common 3.0).

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 Datastream / DataComponent schemas | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms `updatable` is a normative property on scalar `DataComponent` subclasses per the bundled schemas. **Primary citation.** |
| **OGC 23-011r1** — OGC SWE Common 3.0 | "OGC SWE Common 3.0 (OGC 23-011r1)" under *OGC Standards* (verified at [`../references.md`](../references.md) line 152) | Authority for the meaning of the `updatable` flag on a DataComponent (whether the value/definition may be changed after creation). |
| **OGC 23-001** — CSAPI Part 1 / update-path conformance | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms PUT/PATCH conformance must honor the schema declarations the server itself round-trips on GET. The server is responsible for honoring contracts it advertises. |
| **RFC 7807** §3 (problem-detail "identify the problem") | "RFC 7807 — Problem Details for HTTP APIs" under *IETF RFCs* | Supporting: when enforcement IS added, the rejection should be specific (`"component <path> is declared updatable=false"`), not a generic 400. |

**Out-of-scope sources (not cited):** OAS 3.0.3, JSON Schema 2020-12, RFC 7493.

> **References-list verification.** Plan §7 speculated SWE Common as
> "OGC 12-000". Curated entry confirmed as **OGC 23-011r1** at
> [`../references.md`](../references.md) line 152. No
> reference-list gap.

## 5. Alternatives considered (internal-only)

Only one viable shape — pre-update diff against existing schema,
reject on any change to a component declared `updatable=false`.

### Option A — Pre-update diff with `updatable=false` gating (recommended)

Pseudocode for `UpdateDatastreamSchema` (analogous in
`UpdateControlStreamSchema`):

```go
func (h *DatastreamHandler) UpdateDatastreamSchema(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "dataStreamId")

    var newSchema domains.DatastreamSchema
    if err := render.DecodeJSON(r.Body, &newSchema); err != nil {
        ...400...
    }

    existing, err := h.repo.GetSchema(id)
    if err != nil { ...404 / 500... }

    if violations := diffNonUpdatable(existing, &newSchema); len(violations) > 0 {
        render.Status(r, http.StatusConflict)   // or 400
        render.JSON(w, r, map[string]any{
            "error": "edit rejected: components declared updatable=false were modified",
            "violations": violations,           // list of component paths
        })
        return
    }

    if err := h.repo.UpdateSchema(id, &newSchema); err != nil { ...500... }
    w.WriteHeader(http.StatusNoContent)
}
```

`diffNonUpdatable(existing, new) []string` walks both schemas in
parallel; for each leaf-scalar component on `existing` where
`Updatable != nil && *Updatable == false`, compares the
corresponding component on `new`; returns the list of component
paths whose definition or value changed.

- Two block edits (one per handler).
- Mirrors parent #21's "consult model-side metadata; reject with a
  specific message" shape.
- Risk: the diff helper is non-trivial (parallel schema walk,
  component-path tracking). Per plan §8 Q1, this report defers
  line-level implementation to the fix PR. The issue body
  describes the shape (pre-update fetch + diff + path-list
  rejection), not the implementation.

**Lead with Option A only.** No alternatives considered.

## 6. Recommended fix

**Option A.** At each schema-update handler:

1. Fetch the existing schema before applying the requested edit.
2. Walk the existing and incoming schemas in parallel; collect
   the paths of any components where the existing component has
   `Updatable != nil && *Updatable == false` AND the incoming
   component differs (by definition or value).
3. If the violations list is non-empty, return 409 Conflict (or
   400) with a specific RFC 7807 message naming the violated
   component paths.
4. Apply this gating in **both**:
   - `internal/api/datastream_handler.go:233`
     (`UpdateDatastreamSchema`)
   - `internal/api/control_stream_handler.go:246`
     (`UpdateControlStreamSchema`)
5. The resource-level `UpdateDatastream`
   (`datastream_handler.go:159`) should apply the same gating if
   its body can include schema mutations.

Implementation surface: one new helper (schema-diff walker), one
helper call + rejection block at each of two-or-three update
handlers. No new types, no model changes, no schema migration.

> **Note on `updatable` semantics.** Plan §8 Q2 raised two
> readings: (a) schema-level immutability (component
> *definition* cannot change), or (b) data-level immutability
> (recorded *value* cannot change). The fork's update handlers
> work on schemas (not individual observations), so reading (a)
> is the natural fit for these handler sites. If reading (b) is
> the intended SWE Common interpretation, separate enforcement
> on the observation POST path would be needed — let the
> maintainer pick the conformance interpretation; this filing
> recommends reading (a) as the obvious server-enforcement
> default and notes both readings.

## 7. Scope guard

What NOT to touch as part of this filing:

- POST-time validator (`observation_schema_validation.go`) —
  parent #21 fix in place; no change.
- `Constraint` enforcement — separate filing, report-09.
- Model layer — `Updatable` is correctly declared and
  round-trips; no model changes.
- Schema migration — no DB changes.
- The `updateable` vs `updatable` JSON-tag inconsistency in
  `characteristics.go:96` vs `datastream.go:223` — adjacent
  finding (§2); recommend separate followup if maintainer wants
  it tracked.

## 8. Fork-side context (internal-only)

- This is the **third filing** from issue #21's audit:
  - `NilValues` — parent, closed by upstream commit adding
    `matchesNilValue`.
  - `Constraint` — sibling, report-09 (`90f2294`).
  - `Updatable` — this filing.
- Distinguishing feature from #21 and report-09: the fix site is
  **PUT/PATCH update handlers**, not the POST-time validator. Two
  schema-update handlers are in scope; the resource-level PUT may
  also need gating.
- The adjacent JSON-tag-typo finding (`updateable` vs
  `updatable`) is **deliberately not bundled** — it's a separate
  defect class (interface inconsistency / wire-format) and
  warrants its own filing if at all.
- No fork-side patch; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Affected handler scope | **Three handlers**: `UpdateDatastreamSchema` (datastream_handler.go:233), `UpdateControlStreamSchema` (control_stream_handler.go:246), and `UpdateDatastream` (datastream_handler.go:159) if its payload can include schema. The two schema-update handlers are the primary fix sites. |
| `updatable=false` semantics: schema vs data immutability? | **Lead with schema-level (reading a)** for these handler sites; both `Update*Schema` handlers operate on the schema resource, so this is the natural fit. Note both readings in the issue body and let the maintainer choose for the observation-POST path if they prefer reading (b). |
| Bundle with report-09 (`Constraint`)? | **No.** Eval recommended separate filings; affected handler surface is different. Cross-link via "see related" footer. |
| Severity P3 vs P2? | **P3 retained.** Update-only surface; recoverable via subsequent GET; client-visible error behavior. |
| Live reproducer required? | **No.** Auth-gated multi-step sequence; defect is structural (zero `Updatable` refs in `internal/api/`, plus bare schema-update handler bodies). Static + parent #21 transitive sufficient. |
| Cite eval/evidence paths? | **Yes**, validation-chain footer. |
| References-list canonical entry for SWE Common | **OGC 23-011r1** (verified at [`../references.md`](../references.md) line 152). |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P3] DatastreamDataComponent.Updatable orphaned on update path — third sibling of issue #21`

**Labels:** `bug`, `spec-conformance`

---

### Context

Issue #21 (closed) added `matchesNilValue` to the POST-time
observation validator, closing the gap where
`DatastreamDataComponent.NilValues` was model-declared and
GET-round-tripped but never consulted by the validator.

This is the **third filing** from #21's audit (after `Constraint`
in the sibling filing). Same shape — model-declared,
GET-round-tripped, but never consulted — except the affected
handler surface is the **PUT/PATCH schema-update path**, not the
POST-time validator. `git grep` shows zero references to
`Updatable` in the entire `internal/api/` package, and the schema
update handlers perform a bare `DecodeJSON → UpdateSchema → 204`
with no diff or gating.

Verified on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

A Datastream (or ControlStream) DataComponent declared
`updatable: false` can still be edited via the schema-update
handler. The change is silently persisted; subsequent GET reflects
the new value with no rejection.

### Static evidence

`Updatable` reference audit (HEAD `df6da0d`):

```text
$ git grep -nE 'Updatable' upstream/main -- internal/api/
(zero matches)

$ git grep -nE 'Updatable' upstream/main -- internal/model/
internal/model/common_shared/characteristics.go:96:  Updatable *bool `json:"updateable,omitempty"`
internal/model/domains/datastream.go:223:           Updatable *bool `json:"updatable,omitempty"`
```

The schema-update handler bodies are bare:

```go
// internal/api/datastream_handler.go:233
func (h *DatastreamHandler) UpdateDatastreamSchema(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "dataStreamId")

    var schema domains.DatastreamSchema
    if err := render.DecodeJSON(r.Body, &schema); err != nil {
        ...400...
    }
    if err := h.repo.UpdateSchema(id, &schema); err != nil {
        ...500...
    }
    w.WriteHeader(http.StatusNoContent)
}
```

`UpdateControlStreamSchema` at
`internal/api/control_stream_handler.go:246` has the identical
shape. Neither fetches the existing schema, diffs against it, nor
consults `component.Updatable` on any component.

### Asymmetry table

| Aspect | `NilValues` (post-#21) | `Constraint` (sibling) | `Updatable` (this filing) |
|---|---|---|---|
| Model-declared | ✓ | ✓ | ✓ |
| GET round-trip | ✓ | ✓ | ✓ |
| Handler consultation | ✓ (POST validator) | ✗ (POST validator) | ✗ (PUT/PATCH update) |
| Fix site | `validateDataComponentValue` | `validateDataComponentValue` | `Update*Schema` handlers |

### Adjacent finding (not bundled, FYI)

The two model declarations of `Updatable` use **different JSON tag
spellings**:

```text
internal/model/common_shared/characteristics.go:96:  json:"updateable,omitempty"   ← typo (extra e)
internal/model/domains/datastream.go:223:            json:"updatable,omitempty"    ← canonical SWE Common
```

This is a separate wire-format defect (likely client-incompatible
across resource families); recommend filing separately if
desired.

### Live evidence

Not included in this filing — POST a constrained-schema
Datastream then PUT/PATCH it requires authenticated admin
endpoints. The defect is structural (zero `Updatable` references
in the validator package; bare update-handler bodies); behavior
follows deterministically. Parent issue #21's pre-fix matrix
demonstrates the same model-vs-handler asymmetry pattern.

### Recommended fix

At each schema-update handler, fetch the existing schema before
applying the requested edit, diff it against the incoming schema,
and reject (409 Conflict, or 400) if any component declared
`Updatable=false` was modified:

```go
func (h *DatastreamHandler) UpdateDatastreamSchema(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "dataStreamId")

    var newSchema domains.DatastreamSchema
    if err := render.DecodeJSON(r.Body, &newSchema); err != nil { ...400... }

    existing, err := h.repo.GetSchema(id)
    if err != nil { ...404 / 500... }

    if violations := diffNonUpdatable(existing, &newSchema); len(violations) > 0 {
        render.Status(r, http.StatusConflict)
        render.JSON(w, r, map[string]any{
            "error":      "edit rejected: components declared updatable=false were modified",
            "violations": violations, // list of component paths
        })
        return
    }

    if err := h.repo.UpdateSchema(id, &newSchema); err != nil { ...500... }
    w.WriteHeader(http.StatusNoContent)
}
```

Apply identically in `UpdateControlStreamSchema`. If
`UpdateDatastream` (resource-level PUT) accepts schema mutations
in its body, gate there too. Implementation surface: one
schema-diff helper plus one rejection block per handler. No new
types, no model changes, no schema migration.

**Note on semantics.** SWE Common's `updatable` flag admits two
readings: (a) the component's *definition* cannot change
(schema-level immutability), and (b) recorded *values* for that
component cannot change (data-level immutability). The schema-update
handlers naturally fit reading (a); we recommend leading with that
interpretation and let the maintainer decide whether reading (b)
warrants separate enforcement on the observation-POST path.

### Spec authority

- **OGC 23-002** CSAPI Part 2 — `updatable` is a normative
  property on scalar `DataComponent` subclasses. Primary
  citation.
- **OGC 23-011r1** SWE Common 3.0 — defines the `updatable` flag
  semantics on DataComponent.
- **OGC 23-001** CSAPI Part 1 — PUT/PATCH conformance must honor
  schema declarations the server itself round-trips on GET.
- **RFC 7807** §3 — rejection messages should identify the
  specific violation (component path, declared flag).

### Severity

**P3** — narrower surface than the `Constraint` filing
(update-only path; recoverable via subsequent GET). No
data-corruption risk for existing observations; the failure mode
is "client edits a non-updatable schema component and the change
silently sticks." Spec-conformance defect.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-021.md`](../issue-evaluations/issue-021.md) §"Adjacent finding"
- Evidence (static): [`docs/research/evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md)
- Evidence (live, parent matrix): [`docs/research/evidence/issue-021/live-test-2026-04-30.md`](../evidence/issue-021/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-021/spec-authority-2026-04-30.md`](../evidence/issue-021/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-10-updatable-orphaned-in-validator.md`](../upstream-issues/plan-10-updatable-orphaned-in-validator.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #13

**See also:** sibling filing for `Constraint`
(report-09 / `90f2294`).
