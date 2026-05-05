# Research Plan 10 — `DatastreamDataComponent.Updatable` orphaned in validator

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
item **13**, copied verbatim:

> ### 13. `DatastreamDataComponent.Updatable` orphaned in validator
> - **Source:** Issue #21 closure (adjacent finding, scoped out by eval).
> - **Category:** Defect (P3).
> - **Summary:** `DatastreamDataComponent.Updatable` round-trips on GET but
>   is not consulted by the validator on the PUT/PATCH path. Same shape as
>   Constraint; cross-cutting check needed on the update handlers.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P3**, parent #21 was P2,
> sibling `Constraint` is P2. `Updatable` is one severity tier
> lower because (a) the affected surface is narrower (PUT/PATCH
> only, not POST), and (b) the practical impact is "client edits
> a non-updatable field and gets silent acceptance" — recoverable
> via subsequent GET, unlike a constraint violation that may
> propagate downstream into analyses. P3 stands.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#21` — closed by upstream commit(s)
  applying the `nilValues` consultation pattern. This filing is
  the **`Updatable` sibling** explicitly scoped out of #21 per the
  eval's recommendation, paired with `Constraint` (plan-09).
- This filing is sibling to **plan-09** (`Constraint`) but scoped
  separately because the affected handler surface is different
  (update path vs. POST validation).
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-021.md`](../issue-evaluations/issue-021.md) | Full evaluation. "Adjacent finding (not in original body) — sibling fields are similarly orphaned" §confirms `git grep` for `Updatable` in `internal/api/` returns zero matches; recommends separate filing. |
| [`../evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md) | Source for the validator-orphan grep methodology. The same shape applies to `Updatable`: model declared, JSON-tagged, GET-round-trip-preserved, **never** consulted by update-path validation. |
| [`../evidence/issue-021/live-test-2026-04-30.md`](../evidence/issue-021/live-test-2026-04-30.md) | Pre-#21-fix matrix is for `nilValues`. **A fresh `Updatable`-specific matrix must be captured during Step 4.2** — declare a Datastream with a DataComponent marked `Updatable: false`, then PATCH/PUT a different value at that component path; current behavior is silent acceptance, expected behavior is rejection. |
| [`../evidence/issue-021/spec-authority-2026-04-30.md`](../evidence/issue-021/spec-authority-2026-04-30.md) | OAS31 + SWE Common normative shape for scalar component types. The `updatable` flag is declared on every scalar component; non-enforcement on the update path is a CSAPI conformance gap (server is responsible for honoring the contract its own GET response declared). |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted #21 (P2 spec
conformance) and shipped the validator extension for `nilValues`.
Implications:

- Tone: **the third of three audit findings the #21 evaluation
  flagged** (`nilValues` shipped; `Constraint` is plan-09; this is
  `Updatable`). Frame as "the third sibling from the same audit;
  affects a different handler surface (update path) so the fix
  site is different from #21 and plan-09's filing."
- The fix touches **PUT/PATCH handlers** rather than the POST-time
  validator. This is a *different file* from parent #21's fix and
  plan-09's filing — important to surface clearly so the maintainer
  doesn't expect a one-line edit to the same function.

## 5. Upstream commits relevant to the area

- The commit(s) that closed `#21` — drafter must identify exactly
  via §6. Cite as the **shape precedent** (consult model-side
  metadata field, reject with a specific message), not the **site
  precedent** (this filing's fix site is the update handler).
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) `Updatable` is declared on `DatastreamDataComponent` at the
      model layer and round-trips on GET;
  (b) `Updatable` is **not** consulted anywhere in `internal/api/`;
  (c) the PUT/PATCH handlers for Datastream / DatastreamSchema /
      Observation (whichever applies) do not gate writes by
      `Updatable`.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm Updatable is NOT consulted anywhere in internal/api/:
git grep -nE 'Updatable' upstream/main -- internal/api/

# Confirm Updatable IS declared on DatastreamDataComponent at the model layer:
git show upstream/main:internal/model/domains/datastream.go | Select-String -Pattern 'Updatable' -Context 2,2

# Locate the relevant update handlers (PUT/PATCH on Datastream / DataStreamSchema):
git grep -nE 'func.*Update(Datastream|DataStream|Observation)' upstream/main -- internal/api/

# Read the update handlers to confirm no Updatable gating:
git show upstream/main:internal/api/datastream_handler.go | Select-String -Pattern 'Update|Updatable|PATCH|PUT' -Context 4,4

# Recent history on the relevant files:
git log --oneline upstream/main -- internal/api/datastream_handler.go internal/model/domains/datastream.go | Select-Object -First 10

# Sanity: confirm parent #21 fix shape is in place (validator consults NilValues):
git show upstream/main:internal/api/observation_schema_validation.go | Select-String -Pattern 'NilValues' -Context 2,2
```

```pwsh
# Live re-verification — auth required for POST and PUT/PATCH. Sketch:
# 1. POST a Datastream with a schema declaring a DataComponent with `"updatable": false`.
# 2. GET the Datastream — confirm "updatable": false round-trips.
# 3. PATCH the Datastream's schema to change the component's value or definition.
#    Expected: 400 / 409 with a clear "field is not updatable" message.
#    Current behavior: 200 / 204 (silent acceptance) — defect confirmed.
# 4. GET again — confirm the change was persisted (proves silent acceptance).
```

Expected on unchanged state:
- `Updatable` declared on `DatastreamDataComponent` at the model
  layer.
- `Updatable` not consulted anywhere in `internal/api/`.
- Update handlers accept changes regardless of `Updatable` value.
- Parent #21 fix in place (sanity).

If `Updatable` is already consulted on `upstream/main`, **stop
drafting** — defect already fixed.

## 7. Spec-authority sources

Direct binding via OAS31 / SWE Common.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 Datastream / DataComponent schemas | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms `updatable` is a normative property on scalar `DataComponent` subclasses per the bundled schemas. **Primary citation.** |
| **OGC 12-000** — SWE Common Data Model, `updatable` flag semantics | "OGC SWE Common Data Model 2.0 (OGC 12-000r2)" under *OGC SWE Common Standards* (drafter must verify the exact reference-list entry, same as plan-09) | Authority for the meaning of the `updatable` flag (whether a component value may be changed after creation). |
| **OGC 23-001** — CSAPI Part 1 / update-path conformance | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms PUT/PATCH conformance must honor the schema declarations the server itself round-trips on GET. |
| **RFC 7807** §3 (problem-detail "identify the problem") | "RFC 7807 — Problem Details for HTTP APIs" under *IETF RFCs* | Supporting: when enforcement IS added, the rejection should be specific (`"field <path> is declared updatable=false"`), not a generic 400. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — relevant only as transport for the OGC schemas.
- JSON Schema 2020-12 — covered transitively.
- RFC 7493 — relevant for parent #19 / #20, not for this conformance
  defect.

> **Note on SWE Common reference-list entry.** Same caveat as plan-09:
> drafter must verify the SWE Common entry exists in the curated
> references list. Surface to user if missing.

## 8. Open questions

1. **Affected handler scope.** The fix touches **update handlers**,
   not the POST-time validator. The exact set of handlers affected
   depends on what "update a DataComponent" means in cs-go's URL
   shape — could be `PATCH /datastreams/{id}` (whole-resource
   update including schema), `PATCH /datastreams/{id}/schema`,
   `PUT /datastreams/{id}/schema/components/{path}`, or some
   combination. **Tentative answer: §6 must enumerate the actual
   handler surface; the issue body should describe the
   affected scope based on §6's findings, not hypothesize.**
2. **What does `updatable=false` mean exactly?** Two readings:
   (a) the component's *definition* cannot be changed (schema-level
       immutability);
   (b) observations writing to that component cannot change a
       previously-recorded value (data-level immutability —
       essentially "this component is single-write").
   The spec authority §6 must surface should disambiguate.
   **Tentative answer: lead with reading (a) — schema-level — as
   the obvious server-enforcement interpretation; if SWE Common
   indicates reading (b), note both in the issue body and let the
   maintainer pick the conformance interpretation.**
3. **Bundle with plan-09 (`Constraint`)?** No — eval explicitly
   recommended separate filings, and the affected handler surface
   is different (update path vs. POST validation). Mention plan-09
   as a "see related" footer once filed. **Tentative answer:
   separate filing.**
4. **Severity P3 vs. P2?** Plan retains P3 per backlog. Narrower
   surface (update only), recoverable via subsequent GET. P3
   stands.
5. **Live reproducer required?** Auth needed and the multi-step
   POST→GET→PATCH→GET sequence is non-trivial. Static evidence
   should be sufficient; the conformance argument is structural.
   **Tentative answer: capture live opportunistically; static-only
   acceptable.**
6. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–09.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: parent #21 (closed by `<commit>`) extended `validateDataComponentValue` to consult `nilValues` on the POST path. The eval flagged `Updatable` as a third sibling of the same shape (model-declared, GET-round-tripped, validator-uninvoked) but on the **update-path** rather than the POST-validation path. |
| **Claim** | One bullet: a Datastream DataComponent declared `updatable: false` can still be edited via PUT/PATCH; the value or definition is silently changed and the GET response reflects the new value, with no rejection. |
| **Static evidence** | `git grep -nE 'Updatable' upstream/main -- internal/api/` → zero matches; declaration on `DatastreamDataComponent` at the model layer; round-trip-on-GET evidence. From [`evidence/issue-021/static-analysis-2026-04-30.md`](../evidence/issue-021/static-analysis-2026-04-30.md), refreshed in §6. |
| **Live evidence** | If auth permits: 4-step matrix — POST datastream with `updatable=false` component; GET to confirm round-trip; PATCH to change the component; GET to confirm silent acceptance. If not: cite parent #21's matrix as transitive precedent (same audit shape). |
| **Spec authority** | One paragraph: OGC 23-002 / Part 2 schemas declare `updatable` as a normative property on scalar `DataComponent` subclasses. SWE Common (OGC 12-000) defines the flag's semantics. CSAPI Part 1 update-path conformance requires honoring schema declarations. RFC 7807 §3 supports the rejection-message-specificity requirement. |
| **Recommended fix** | At the relevant update handler(s) (enumerated by §6), before applying the requested edit, consult `component.Updatable` (or its parent component's flag) and reject with a specific 400/409 message if `false`. Mirror parent #21's structural pattern (consult model-side metadata, specific rejection message). Defer line-level implementation to the fix PR. |
| **Scope guard** | "What NOT to touch": no change to the POST-time validator (parent fix in place); no `Constraint` enforcement (separate filing — plan-09); no model-layer changes (already correct); no schema migration. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-10-updatable-orphaned-in-validator.md`](../upstream-issue-reports/report-10-updatable-orphaned-in-validator.md)

Same `NN` and `<slug>`
(`10-updatable-orphaned-in-validator`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
