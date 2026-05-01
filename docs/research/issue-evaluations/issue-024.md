# Issue #24 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#24 |
| Title | [P3 enhancement] Inline `@link.href` should be absolute URI to match spec `format: uri` |
| Labels | `enhancement` |
| Verified HEAD | `08cf161` (formatter + handler files unchanged from cited fork sha) |
| **Verdict** | **KEEP** |
| Severity | **P3 enhancement** — confirmed accurate |

## Summary

The asymmetry the issue describes is real, reproducible, and the body's
spec citation is exact. The current behaviour is not a hard
interoperability break (relative URIs are RFC-3986 base-resolvable),
but it (a) does not match the spec's worked examples (which uniformly
use absolute URIs), (b) is internally inconsistent with the same
server's `links[]` array convention which is already absolute, and
(c) would be flagged as a `format`-vocabulary lint by strict OAS31
validators.

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-024/static-analysis-2026-04-30.md). Two assignment sites build relative hrefs unconditionally:
  - `internal/api/datastream_handler.go:136` → `&Link{Href: "systems/" + systemID}`
  - `internal/api/control_stream_handler.go:145` → same shape
  - Neither calls `formaters.ToFunctionalAssociationHref(...)`, even though that helper is configured with `cfg.API.BaseURL` at startup (`internal/api/router.go:27`) and **is** producing absolute hrefs for the supplementary `links[]` array on the same response.
  - Helper at `internal/model/formaters/association_links.go:371-389` is **idempotent** on already-absolute inputs (short-circuits via `parsed.IsAbs()`), so applying it broadly is safe.
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-024/live-test-2026-04-30.md):
  - T1 reproduces the body's symptom verbatim: `system@link.href = "systems/666ed3fe-..."` (relative) while the same response's `links[]` entry with `rel: ogc-rel:systems` is `"https://129-80-248-53.sslip.io/csapi-go-head/systems/666ed3fe-..."`.
  - T2 confirms user-supplied absolute hrefs round-trip preserved (POST → GET, both absolute) — proves the fix can be applied with no breakage to compliant clients.
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-024/spec-authority-2026-04-30.md):
  - Inline `@link` schema (OAS31 line 312-324) is the OGC Link object with `href: format: uri`, reused via `*ref_11`/`*ref_12` for all enumerated inline link properties.
  - Spec worked examples (lines 1922-1929 Datastream, 3547-3550 Command) uniformly use absolute `https://...` URIs.
  - The schema explicitly distinguishes `format: uri` (used here) from `format: uri-reference` (used elsewhere for XLink-style associations) — the choice of `uri` is intentional in the spec.

## Where the body is correct

- Symptom description verbatim accurate.
- Spec citation (`format: uri` at line 318-322 of the cited yaml) verifies against the bundled OAS31 (lines 312-324).
- Internal-consistency argument (`links[]` already absolute) is verifiable in the same response.
- P3 severity is correct — minor conformance gap, not a break.
- Fix sketch (apply `ToFunctionalAssociationHref(...)` in formatter) is one of two correct fix sites.

## Refinements / additions

1. **Root-cause vs. symptom-site fix sites**: the body suggests fixing in the formatter. The deeper root cause is in the **handlers** which store a relative href to begin with (`datastream_handler.go:136`, `control_stream_handler.go:145`). Recommended approach is **both**:
   - Handler-side fix normalises *storage* so what's persisted matches what's emitted.
   - Formatter-side fix is defence-in-depth in case any other write path bypasses the handler (e.g. legacy data, future GORM hooks).
   - The helper is idempotent on absolute inputs so applying it at both sites is safe.

2. **Spec-strictness nuance** (worth recording so the PR's "OAS schema validation tests" acceptance criterion is calibrated): `format: uri` in OpenAPI 3.1 / JSON Schema 2020-12 is **annotation-only by default**. A lax validator passes relative URIs silently; a strict validator using the format-assertion vocabulary flags them. The body's "may flag" hedge is correct — the project's chosen validator behaviour determines whether existing tests fail today.

3. **Affected-fields completeness** — the body's enumeration matches what `git grep` finds in `internal/model/domains/`. 17 inline link properties total across the 7 enumerated resource types. No omissions detected.

4. **Acceptance criterion sharpening**: the PR should additionally include a test asserting `^https?://` for **every** inline `@link.href` in fixtures of every resource type, not only DS/CS. This catches future regressions on resource types not explicitly in the fix scope (e.g. SamplingFeature, Deployment, System).

5. **Optional companion improvement** — the body mentions "the sibling P4 enhancement" for `Rel`/`Type`/`Title` population. Out of scope for #24 itself but worth tracking that fixing href without populating `rel`/`type`/`title` leaves the link half-rich.

## Recommended fix scope (concrete)

For a PR closing #24:

1. **Patch handler-synthesis sites**:
   ```go
   // internal/api/datastream_handler.go:136
   datastream.SystemLink = &common_shared.Link{
       Href: formaters.ToFunctionalAssociationHref("/systems/" + systemID),
   }
   // internal/api/control_stream_handler.go:145
   cs.SystemLink = &common_shared.Link{
       Href: formaters.ToFunctionalAssociationHref("/systems/" + systemID),
   }
   ```
   Note the leading `/` so the helper produces a path-rooted absolute URL.

2. **Patch JSON formatter sites** to absolutize all inline `@link` properties on read:
   - `internal/model/formaters/json_formatters/datastream_json.go`
   - `internal/model/formaters/json_formatters/control_stream_json.go`
   - Audit other emission paths for Observation/Command/Deployment/System/SamplingFeature (formatter file inventory is sparse — some types emit via direct GORM JSON marshal, in which case a normalising hook on the model `Link` JSON marshaller is an alternative).

3. **Test** — a single shared test helper that round-trips POST→GET on every inline `@link` property for every resource type and asserts each emitted href matches `^https?://`.

## Verdict rationale

KEEP. P3 enhancement is appropriate. The defect is real and minor; the
fix is bounded; the helper to apply is already in place and idempotent;
no client breakage expected. The eval refines the body's fix scope from
"formatter-only" to "handler + formatter" and sharpens the acceptance
criterion to an `^https?://` regex assertion across all resource types.
