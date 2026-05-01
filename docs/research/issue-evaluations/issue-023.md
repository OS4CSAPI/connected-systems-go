# Issue #23 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#23 |
| Title | [P3 docs] Document result-schema validation contract: required/optional, null semantics, `nilValues`, open-schema policy |
| Verified HEAD | `08cf161` (validator unchanged from cited fork sha) |
| **Verdict** | **KEEP — docs deliverable, all 5 claims verified** |
| Severity | P3 (docs hygiene) — accurate as filed |

## Summary

Pure documentation issue. All 5 enumerated claims in the body are
factually correct on HEAD `08cf161`, and none of the described
behaviours are documented anywhere in the repository. The proposed
deliverable (a single page `docs/api/observation-result-schema.md`) is
appropriate. The dependency note ("depends on #21 and #22 landing
first") is correct, with one refinement: per the #22 evaluation, the
doc should reflect the **reframed** #22 outcome (current null-rejection
is intentional and spec-conformant; document omission and `nilValues`
as the canonical absence carriers, not "null is now allowed").

## Verification

- **Static + claim-by-claim** — see [static-analysis-2026-04-30.md](./evidence/issue-023/static-analysis-2026-04-30.md). Each of the 5 claims verified against the source file:
  1. optional ≠ null (cross-ref #22 ✓)
  2. nilValues plumbed but not consumed (cross-ref #21 ✓)
  3. Open schema (re-tested fresh — see live evidence)
  4. `isNumber` is `_, ok := v.(float64); return ok` — float64-only (lines 341-344)
  5. Error format is positional dotted path + free-form English (no machine separator)
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-023/live-test-2026-04-30.md). Open-schema test: undeclared `undeclared_extra_field` and `another_typo_field` keys → 201 → round-trip GET preserves both verbatim. Confirms typos are indistinguishable from extensions.
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-023/spec-authority-2026-04-30.md). Open-schema is neither mandated nor prohibited (defensible implementation choice). RFC 7807 `application/problem+json` is the OGC-API-conventional error shape and the strongest recommendation for the doc deliverable.
- **Repo audit** — `git ls-files docs/api/` returns nothing (directory does not exist). `git grep -l -i "nilValues|optional.*omit|extra.*field" -- docs/ README.md` returns only our research evidence — no user-facing docs exist on these topics.

## Where the body is correct

- All 5 claims are factually verifiable.
- The "no docs exist" implicit premise is verified (no `docs/api/` directory; only research evidence references the topics).
- Dependency ordering on #21 and #22 is sound — docs should reflect the post-fix world.
- The list of topics to cover is comprehensive for a single-page reference.

## Refinements / additions to the body's deliverable

1. **Reflect #22's reframed outcome**, not its body's proposed fix. The doc should state: omission is the canonical "no value" carrier for `optional: true` fields; `nilValues` strings are the canonical carrier for "value present, but represents missing/out-of-range with a known reason"; JSON `null` is **not accepted** and produces 400. This contradicts the body's "after #22 lands" framing if #22 is accepted as a doc/diagnostics improvement rather than a code-relaxation fix.

2. **Open-schema policy decision needs a separate decision step before docs**. The body says "either tighten it (reject) or formalize it (accept-with-warn-link), then document." Recommend an explicit decision recorded in the doc. The live test makes the data-quality risk of typos concrete: if the project chooses to retain silent-open behaviour, the doc should at minimum warn users to lint their schemas client-side.

3. **Numeric type model — additional caveat worth documenting**: a future change to `json.Decoder.UseNumber()` would silently break `isNumber`/`isIntegerNumber` (which type-assert on `float64`). Worth a "stability note" callout in the doc.

4. **Error-shape suggestion — align on RFC 7807**, not the body's ad-hoc `{error,field,reason,expected}` shape. RFC 7807 `application/problem+json` is the OGC API convention. Migrating to it is potentially a separate code issue worth filing (it changes the response content-type and field names — a breaking change).

## Adjacent finding (not in body) — typo-vs-extension data-quality risk

The live test surfaces a concrete operational risk: an optional-field
typo will silently succeed with no value written for the intended
field, and no signal to the client. This is mentioned in the body's
claim 3 only abstractly. The doc deliverable should include this as a
worked example with a concrete typo case to make the risk vivid.

## Severity confirmation

P3 (docs hygiene) is correct. This is not a code defect; it is a
documentation gap. The only reason it cannot be done immediately is
the dependency on #21 (which adds nilValues consumption) and #22
(which decides the null-handling story).

## Verdict rationale

KEEP. All claims verified, deliverable is appropriate, dependencies are
correctly ordered. The eval refines the deliverable scope: align on
RFC 7807 errors; reflect #22's reframed outcome; explicitly decide and
document the open-schema policy; add a stability note on the numeric
type model. No code change is required for this issue itself; the
research evidence here suffices to inform the doc author when #21 and
#22 close.
