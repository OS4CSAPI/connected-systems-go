# Research Plan 09 — `DatastreamDataComponent.Constraint` orphaned in validator

> **🛑 Mandatory pre-work — read every session before touching this plan or
> its corresponding report:**
>
> 1. Open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm the spec/standard sources cited below match the
>    canonical entries there. **Do not self-source references.** If a
>    citation is needed that is not in that list, stop and surface the
>    gap to the user before proceeding.
> 2. Re-read this entire plan file. Do not work from memory.
> 3. Re-read the source eval and evidence files linked in §3 before
>    drafting the report.

---

## 1. Backlog entry

From [`../upstream-followup-backlog.md`](../upstream-followup-backlog.md)
item **12**, copied verbatim:

> ### 12. `DatastreamDataComponent.Constraint` orphaned in validator
> - **Source:** Issue #21 closure (adjacent finding, scoped out by eval).
> - **Category:** Defect (P2 — spec conformance).
> - **Summary:** `DatastreamDataComponent.Constraint` round-trips on GET but
>   is not consulted by `validateDataComponentValue` in
>   `internal/api/observation_schema_validation.go`. Clients declaring
>   numeric constraints (min/max, allowed-values list) get no enforcement on
>   observation POST. Same "model declares it, GET round-trips it, validator
>   ignores it" shape that #21 closed for `nilValues`.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P2 — spec conformance**. Eval
> (issue-021 "Adjacent finding") explicitly identifies `Constraint`
> as a *separate filing* of the same shape as #21 (which was P2).
> P2 stands.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#21` — closed by upstream commit(s)
  applying the `nilValues` consultation pattern to
  `validateDataComponentValue`. This filing is the **`Constraint`
  sibling** explicitly scoped out of #21 per the eval's
  recommendation to file independent defects of the same shape as
  separate issues.
- Backlog item 13 (plan-10) is the symmetric `Updatable` filing;
  same shape, separate scope, separate filing.
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-021.md`](../issue-evaluations/issue-021.md) | Full evaluation. "Adjacent finding (not in original body) — sibling fields are similarly orphaned" §confirms `git grep` for `Constraint` in `internal/api/` returns zero matches; explicitly recommends filing as a separate issue. |
| [`../evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md) | Source for the validator-orphan grep methodology and the model-layer plumbing confirmation. The same shape applies to `Constraint`: model declared, JSON-tagged, GET-round-trip-verified, validator-uninvoked. |
| [`../evidence/issue-021/live-test-2026-04-30.md`](../evidence/issue-021/live-test-2026-04-30.md) | Pre-#21-fix matrix is for `nilValues`. **A fresh `Constraint`-specific matrix must be captured during Step 4.2** — declare a Datastream with `Constraint: {min: 0, max: 100}` (or allowed-values list per CSAPI/SWE Common shape), POST observations with values inside and outside the constraint, confirm the validator does not differentiate. |
| [`../evidence/issue-021/spec-authority-2026-04-30.md`](../evidence/issue-021/spec-authority-2026-04-30.md) | OAS31 + SWE Common normative shape for scalar component types. The `constraint` property is declared at the SWE Common level on every `DataComponent` subclass that supports numeric/categorical constraints; non-enforcement is a CSAPI conformance gap. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted #21 (P2 spec
conformance) and shipped the validator extension for `nilValues`.
Implications:

- Tone: **applying the same pattern to a sibling field of the same
  shape**. Frame as "this is the second of three audit findings the
  #21 evaluation flagged (`Constraint` and `Updatable` were
  explicitly scoped out into separate filings); same code site,
  same fix shape."
- The fix shape is unambiguous in structure, but **the comparison
  semantics are different** from `nilValues`: `Constraint` is a
  range/enumeration object, not a discrete-value list. Drafter
  must articulate this in the recommended fix.

## 5. Upstream commits relevant to the area

- The commit(s) that closed `#21` — drafter must identify exactly
  via §6. Cite as the precedent for the validator-extension shape
  (consult a model-side metadata field at each leaf-scalar branch
  before rejecting).
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) `validateDataComponentValue` now consults `NilValues`
      (parent fix in place);
  (b) `Constraint` is declared on `DatastreamDataComponent` at the
      model layer and round-trips on GET;
  (c) `validateDataComponentValue` does **not** consult `Constraint`
      at any leaf-scalar branch.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm parent #21 fix shape (validator consults NilValues now):
git show upstream/main:internal/api/observation_schema_validation.go | Select-String -Pattern 'NilValues|nilValues' -Context 2,2

# Confirm Constraint is NOT consulted in the validator:
git grep -nE 'Constraint' upstream/main -- internal/api/

# Confirm Constraint IS declared on DatastreamDataComponent at the model layer:
git show upstream/main:internal/model/domains/datastream.go | Select-String -Pattern 'Constraint' -Context 2,2

# Confirm GET round-trips Constraint (sanity check, model-layer):
git grep -nE 'Constraint' upstream/main -- internal/model/domains/

# Recent history on observation_schema_validation.go (find the closing commit):
git log --oneline upstream/main -- internal/api/observation_schema_validation.go | Select-Object -First 10

# Find SWE Common Constraint shape definition (for fix-shape design):
git show upstream/main:internal/model/domains/datastream.go | Select-String -Pattern 'type.*Constraint|AllowedValues|AllowedTokens|min|max' -Context 3,3
```

```pwsh
# Live re-verification — auth required for POST. Sketch:
# 1. POST a Datastream with a DataComponent declaring:
#    "constraint": {"interval": [[0, 100]]}  (or AllowedTokens for category)
# 2. POST an Observation with result inside the interval (e.g. 50) → 201 (control)
# 3. POST an Observation with result outside (e.g. -5 or 150) → expected 400 (with constraint enforcement)
#    Actual current behavior: 201 (no enforcement) — this is the defect
# 4. GET the datastream → confirm "constraint" round-trips
```

Expected on unchanged state:
- Validator file consults `NilValues` (parent fix in place).
- `Constraint` declared on `DatastreamDataComponent` at the model layer.
- `Constraint` not consulted anywhere in `internal/api/`.
- Live: out-of-constraint values are accepted with 201.

If `Constraint` is already consulted on `upstream/main`, **stop
drafting** — defect already fixed.

## 7. Spec-authority sources

Direct binding via OAS31 / SWE Common Constraint shape definition.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 Datastream / DataComponent schemas (Part 2 OpenAPI bundle) | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms `constraint` is a normative property on every scalar `DataComponent` subclass per the bundled schemas. **Primary citation.** |
| **OGC 12-000** — SWE Common Data Model, AllowedValues / AllowedTokens / AllowedTimes | "OGC SWE Common Data Model 2.0 (OGC 12-000r2)" under *OGC SWE Common Standards* (or equivalent entry — drafter must verify the exact reference-list entry) | Authority for the constraint shape (`AllowedValues` for numeric, `AllowedTokens` for categorical, `AllowedTimes` for temporal). Cite to justify the comparison semantics in the recommended fix. |
| **OGC 23-001** — CSAPI Part 1 / Observation conformance class | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms observation submission must conform to the datastream's declared schema; constraint enforcement is part of that conformance. |
| **RFC 7807** §3 (problem-detail "identify the problem") | "RFC 7807 — Problem Details for HTTP APIs" under *IETF RFCs* | Supporting: when constraint enforcement IS added, the rejection message should be specific (`"value <V> outside declared interval [<min>, <max>]"`), not generic. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — relevant only as transport for the OGC schemas; the
  binding is the OGC schema content, not OAS structural rules.
- JSON Schema 2020-12 — covered transitively via the OGC schema
  citation.
- RFC 7493 — relevant for parent #19 / #20, not for this conformance
  defect.

> **Note on SWE Common reference-list entry.** Drafter must verify
> the exact entry in the curated references list. If SWE Common is
> not present in the curated list, surface the gap to the user
> before drafting the report — the constraint shape semantics
> require a SWE Common citation, and self-sourcing is forbidden.

## 8. Open questions

1. **Comparison semantics** are non-trivial. Unlike `nilValues`
   (discrete-value match), `Constraint` is a structured object
   that may contain `AllowedValues` (intervals + discrete values),
   `AllowedTokens` (string enum), or `AllowedTimes` depending on
   the leaf-scalar component type. **Tentative answer: defer the
   detailed comparison-semantics design to the fix PR; the issue
   body should articulate the *shape* of what needs to be checked
   per leaf-scalar branch (numeric → AllowedValues; category →
   AllowedTokens; time → AllowedTimes), not specify the
   implementation in line-level detail.**
2. **Apply uniformly to all 7 leaf-scalar branches?** Yes —
   matches parent #21's uniform-application recommendation. **Tentative
   answer: explicit scope statement in the issue body that the
   fix should mirror parent #21's "all 7 leaf-scalar branches"
   shape.**
3. **Bundle `Updatable` (backlog item 13 / plan-10) into this
   filing?** No — eval explicitly recommended separate filings,
   and `Updatable` is a different concern (PUT/PATCH path
   semantics, not POST validation). Keep this narrow. **Tentative
   answer: file separately, mention `Updatable` as a one-line
   "see related" footer note pointing to the upstream issue
   number once plan-10 is filed.**
4. **Severity P2 vs. P3?** Plan retains P2 per backlog and eval.
   Spec conformance, observation correctness — direct functional
   impact on clients declaring constraints. P2 stands.
5. **Live reproducer required?** Auth needed for POST and for
   declaring a constrained Datastream. Static evidence + parent
   #21's matrix as transitive precedent (same shape) should be
   sufficient. **Tentative answer: capture live opportunistically;
   static-only acceptable.**
6. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–08.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: parent #21 closed by `<commit>` extended `validateDataComponentValue` to consult `nilValues` at each leaf-scalar branch. The eval flagged `Constraint` as a sibling field of the same shape (model-declared, GET-round-tripped, validator-uninvoked) and explicitly scoped it out into a separate filing. |
| **Claim** | One bullet: clients declaring constraints (`AllowedValues` numeric intervals/lists, `AllowedTokens` string enum, `AllowedTimes` ranges) on a Datastream's DataComponent receive no enforcement on observation POST; out-of-constraint values are silently accepted with 201. |
| **Static evidence** | `git grep -nE 'Constraint' upstream/main -- internal/api/` zero matches; `git grep` shows it declared on `DatastreamDataComponent` at the model layer; round-trips on GET. From [`evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md), refreshed in §6. |
| **Live evidence** | If auth permits: 4-test matrix mirroring #21's structure — declare constrained Datastream, POST in-bounds (201, control), POST out-of-bounds (currently 201, defect), GET round-trip confirms constraint preserved. If not: cite parent #21's matrix as transitive precedent. |
| **Spec authority** | One paragraph: OGC 23-002 / Part 2 schemas declare `constraint` as a normative property on scalar `DataComponent` subclasses. SWE Common (OGC 12-000) defines the shape. CSAPI Part 1 requires observations conform to the datastream's declared schema, which includes constraint. RFC 7807 §3 supports the rejection-message-specificity requirement when the fix lands. |
| **Recommended fix** | Mirror parent #21's structural pattern: at each of the 7 leaf-scalar branches, after the type check passes, consult `component.Constraint` if present and reject with a specific message if the value is out-of-constraint. Comparison semantics dispatch by component type (numeric → AllowedValues; category → AllowedTokens; time → AllowedTimes). Defer line-level implementation to the fix PR. |
| **Scope guard** | "What NOT to touch": no change to `nilValues` consultation (parent fix in place); no `Updatable` enforcement (separate filing — plan-10); no model-layer changes (already correct); no schema migration. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-09-constraint-orphaned-in-validator.md`](../upstream-issue-reports/report-09-constraint-orphaned-in-validator.md)

Same `NN` and `<slug>`
(`09-constraint-orphaned-in-validator`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
