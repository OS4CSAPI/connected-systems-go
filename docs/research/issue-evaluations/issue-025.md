# Issue #25 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#25 |
| Title | [P4 enhancement] Populate `rel`/`type`/`title`/`uid` on inline `@link` objects |
| Labels | `enhancement` |
| Verified HEAD | `5f079c3` |
| **Verdict** | **KEEP** |
| Severity | **P4 enhancement** — confirmed |

## Summary

A purely additive UX enhancement, fully spec-compliant, with no risk to
existing clients. Body's claims verify exactly:

- Struct supports `Href`/`Rel`/`Type`/`Title`/`UID` (`links.go:35-41`).
- Server-synthesized inline `@link` populates **only** `Href`
  (`datastream_handler.go:136`, `control_stream_handler.go:145`,
  `datastream_repository.go:102`, `control_stream_repository.go:102`).
- Supplementary `links[]` builder populates `Href`+`Rel` only
  (`datastream_json.go:63-83`).
- Spec (OAS31 lines 312-372) defines all enumerated optionals exactly as
  body claims — `href` required, others optional.
- Round-trip already preserves client-supplied optional fields end-to-end
  (T2: dsId `16f5115b-4c90-49df-8db9-e0c06e261e5d`).

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-025/static-analysis-2026-04-30.md).
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-025/live-test-2026-04-30.md). Inline `system@link` carries only `href`; supplementary `links[]` carries `href`+`rel`. Optional fields supplied by client persist verbatim through POST → JSONB → GET.
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-025/spec-authority-2026-04-30.md). All listed optional fields are spec-defined and zero are required-when-known.

## Where the body is correct

- Symptom reproduces verbatim.
- Spec citation (lines 318-380 of cited yaml ≈ lines 312-372 of bundled
  OAS31) is exact.
- Affected-formatter list matches `git grep` results.
- P4 severity is correct (no conformance issue).
- Suggested fields and their values are reasonable.

## Refinements

1. **`appendDatastreamAssociationLinks` is misattributed**. The body says it
   "populates only `Href`". That helper actually populates `Href`+`Rel` and
   builds the **supplementary `links[]` array**, not the inline `@link`. The
   inline `@link` is populated at the **handler/repository synthesis sites**
   listed in static-analysis. The functional concern (optional metadata
   missing) applies to both arms — but be precise about which file/function
   to patch:
   - **Inline `@link`** fix sites: `datastream_handler.go:136`,
     `control_stream_handler.go:145`,
     `datastream_repository.go:102`,
     `control_stream_repository.go:102`.
   - **Supplementary `links[]`** fix sites:
     `datastream_json.go:appendDatastreamAssociationLinks` and the parallel
     helper for ControlStream.

2. **Practical caveat on `Title`/`UID` enrichment**. At handler-synthesis
   time (POST), only the `systemID` (path parameter) and `kindID`
   (procedure derivation) are available — not the system's display name or
   uid. To populate `Title`/`UID`, the cleanest place is a **read-time
   enrichment hook** in the JSON formatter that loads minimal system
   metadata (name/uid) from the repository when emitting. Suggested split:
   - **Cheap fields** (`Type` constants, `Rel` constants) — populate at
     synthesis time. Zero extra DB cost.
   - **Expensive fields** (`Title`, `UID`) — populate in formatter via a
     batched preload (one query per page of items, joining systems by
     `SystemID`). Avoids N+1.

3. **Consistency with `links[]` rel naming**. Body suggests `rel="parent"`
   for `system@link`. The cs-go `links[]` array already uses
   `ogc-rel:systems` for the same logical relation. To keep one
   relation-vocabulary in the response, recommend either:
   - omit `rel` from inline `@link` (the property name conveys the
     relation), **OR**
   - use `ogc-rel:host` / matching OGC convention rather than IANA
     `parent`.
   Recording this so the PR aligns rel naming consciously.

4. **`Type` value calibration**:
   - `system@link → type: application/geo+json` ✅ (System is a GeoJSON
     Feature in OGC API CSAPI Part 1).
   - `procedure@link → type: application/sml+json` — **only** when
     served. cs-go also supports `application/json` for procedures.
     Suggest using `Accept`-aware logic or a sensible default
     (`application/sml+json` is correct per spec preference). Worth noting
     in the PR.
   - `deployment@link → type: application/geo+json` (same as system).
   - `samplingFeature@link → type: application/geo+json`.
   - `featureOfInterest@link → type: application/geo+json`.
   - The body only enumerates DS+procedure; the PR should cover all 17
     inline link properties consistently.

5. **Acceptance criteria sharpening**:
   - The third checkbox ("round-trip POST → GET preserves at least the
     fields the client supplied") is **already satisfied** today (T2 in
     live-test) — the improvement scope is strictly *server-side
     enrichment of server-synthesized links*. Worth noting so reviewers
     don't ask for new round-trip tests that already pass.
   - Add a positive assertion: every server-synthesized inline `@link`
     in a sample fixture set carries at least `Type` (cheap to verify,
     zero DB cost).

## Recommended fix scope

For a PR closing #25 (sequenced after or merged with #24's fix):

1. **Cheap pass (no DB cost)** — at handler/repository synthesis sites and
   the `appendDatastreamAssociationLinks` helper, populate `Type`
   (constant per resource type) and standardise on omitting `Rel` for
   inline `@link` (or pick one OGC rel convention).

2. **Enrichment pass (formatter, with batched preload)** — in
   `datastream_json.go:Serialize`/`SerializeAll`, load system metadata
   for the page and populate `Title` (= `system.Name`) and `UID` (=
   `system.UID`) on inline `@link`. Mirror across DS/CS/Observation/
   Command/Deployment/SamplingFeature formatters.

3. **Tests**:
   - Assert `system@link.type == "application/geo+json"` in
     server-synthesized DS responses.
   - Assert `system@link.title` and `.uid` non-empty when the linked
     system has those fields.
   - Assert client-supplied optionals still round-trip (already passes —
     regression guard).

## Verdict rationale

KEEP. P4 enhancement well-grounded. Body's claims verify in code and
spec. Round-trip path already works; only server-side enrichment is
needed. Eval refines the fix scope to (a) precise file/function
targets, (b) cheap-vs-expensive split with batched preload to avoid
N+1, (c) coverage across all 17 inline link properties not just DS+CS,
and (d) `rel`-vocabulary consistency note.
