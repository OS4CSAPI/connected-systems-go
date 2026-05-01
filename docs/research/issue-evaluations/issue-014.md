# Issue #14 evaluation — `GET /api` returns 86/88-byte stub with no paths or schemas

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#14 |
| Title (issue body) | `GET /api returns 86-byte stub with no paths or schemas (no machine-readable service-desc)` |
| Author | Sam-Bolling |
| Labels | `bug`, severity `P2-Important` |
| Repo HEAD evaluated | `629e2c404243a2a34223492da2b5a14a6597b284` |
| Live endpoint | `https://129-80-248-53.sslip.io/csapi-go-head` |
| Evaluation date | 2026-04-30 |
| **Verdict** | **KEEP — valid defect, validated empirically and against spec authority** |

---

## 1. Claim under evaluation

cs-go's `GET /api` endpoint — advertised on the landing page as
`service-desc` with media type
`application/vnd.oai.openapi+json;version=3.0` — returns a hard-coded
86/88-byte JSON literal that contains only `openapi` and `info` fields.
It has no `paths`, no `components`, no `servers`, no description of any
operation. It therefore cannot serve as a machine-readable
service-description for the API. The issue cites
`internal/api/router.go:255–267` and the `// TODO: Implement OpenAPI
3.0 spec generation` comment as proof the implementation is openly
unfinished.

## 2. Evaluation method

Standard six-step rigour pass:

1. Read issue body verbatim. ✅
2. Static analysis of the cited code path and surrounding context. ✅
   See [static-analysis-2026-04-30.md](../evidence/issue-014/static-analysis-2026-04-30.md).
3. Spec-authority research (OAS 3.0.3 root-object fixed fields; OGC
   API – Common Part 1 §9.2 and §11). ✅ See
   [spec-authority-2026-04-30.md](../evidence/issue-014/spec-authority-2026-04-30.md).
4. Live empirical reproduction against HEAD. ✅ See
   [live-test-2026-04-30.md](../evidence/issue-014/live-test-2026-04-30.md).
5. Write up. ← this document.
6. Comment + plan-update + commit + push.

## 3. Findings

### 3.1 Empirical premise: confirmed

`GET /api` on HEAD (`629e2c4`) returns:

```http
HTTP/2 200
content-type: application/vnd.oai.openapi+json;version=3.0
content-length: 88

{"openapi": "3.0.0", "info": {"title": "OGC Connected Systems API", "version": "1.0.0"}}
```

The 2-byte difference vs. the issue body (88 vs. 86) is fully accounted
for by `cfg.API.Title` / `cfg.API.Version` string lengths shifting
between commits; the substance is identical. The cited code at
`router.go:255-267` is byte-for-byte present, including the `// TODO`
comment. There is no `//go:embed` and no static OAS file anywhere in
the repository.

### 3.2 Spec authority: at least one strict `SHALL` is binding

| Requirement | Strength | Bound to cs-go because… | Status |
|---|---|---|---|
| **OAS 3.0.3 §4.7.1.1** — `paths` is **REQUIRED** at the root OpenAPI Object | Hard schema rule | The response declares `"openapi":"3.0.0"` and `Content-Type: application/vnd.oai.openapi+json;version=3.0` (the OAS 3.0 JSON media type per OGC 19-072 §12.2), self-asserting OAS 3.0 conformance | **Violated** |
| **OGC 19-072 `/req/landing-page/api-definition-success` B** — "The content of that response SHALL be an API Definition document." | Strict `SHALL` | cs-go's `/conformance` declares `…/ogcapi-common-1/1.0/conf/landing-page` | **Violated** |
| **OGC 19-072 `/req/landing-page/api-definition-success` C** — "The API Definition document SHALL be consistent with the media type" | Strict `SHALL` | same | **Violated** (Content-Type declares OAS 3.0; body is not valid OAS 3.0) |
| **OGC 19-072 `/rec/landing-page/api-definition-oas`** — if document uses OAS 3.0, it SHOULD conform to the OAS 3.0 requirements class | `SHOULD` | same | **Violated** |

cs-go does **not** declare `/conf/oas30` of OGC API – Common Part 1, so
the stricter `/req/oas30/*` requirements (e.g. `oas-definition-2`,
`completeness`, `oas-impl`) are not directly binding. This is a
narrow precision point relative to the issue's framing (which mentions
"Conformance Class OpenAPI 3.0"), but it does **not** undermine the
defect — the binding `landing-page/api-definition-success` `SHALL` is
sufficient on its own, and the OAS 3.0 spec itself rejects the document
on schema grounds regardless of OGC declarations.

### 3.3 Severity assessment

The issue labels this **P2-Important**. Independent assessment:

- **Functional impact:** any compliant OGC API client that fetches the
  service-description to drive operations (e.g. via openapi-generator,
  Swagger UI, automated CITE testing) will receive a useless document.
  This breaks the primary machine-discoverability path that OGC API
  Common is designed around.
- **Spec impact:** at least one `SHALL` violation against a
  conformance class the server self-declares. CITE conformance testing
  for OGC API – Common Part 1 `landing-page` would fail.
- **Mitigations:** human users can still navigate via the landing page
  (HTML/JSON link traversal), and the API remains usable to clients
  with hard-coded knowledge of CSAPI Part 1 / Part 2 paths. So the
  failure is not "service down".

P2-Important is appropriate. An argument for P1-Critical exists if
formal OGC certification or automated client tooling is on the project
roadmap; the issue's P2 framing is conservative and reasonable.

### 3.4 Suggested-fix evaluation (issue body lists three options)

The issue's three suggested fixes are all valid; brief independent
assessment:

1. **Embed a static OAS bundle via `//go:embed`** — lowest delta,
   highest control over content. Requires authoring/maintaining the
   OAS document, which is non-trivial given the breadth of CSAPI
   Part 1+2 (Features, Systems, Datastreams, Observations, Sampling
   Features, Procedures, plus the EDR/edr-style query envelopes). For a
   hand-curated bundle this is meaningful work but a one-time cost
   (with maintenance churn).
2. **Generate at runtime via `swag` or `chi-openapi`** — lowest
   maintenance delta if the team is willing to add generator
   annotations to handlers. `swag`'s OAS 2 → 3 path is mature; `chi`
   already a dep, so `chi-openapi` is plausible. Risk: generators tend
   to under-describe schemas unless the team disciplines itself with
   tags.
3. **Serve upstream OGC OAS31 documents with patched `servers`** —
   the OGC publishes authoritative reusable OpenAPI building blocks at
   `schemas.opengis.net/ogcapi/...`. Lowest authoring delta. Risk:
   OGC bundles are OpenAPI 3.1; cs-go currently advertises 3.0 media
   type, so either the media type would change to
   `application/vnd.oai.openapi+json;version=3.1` (preferable, modern,
   matches the upstream specs) or selective transcription is needed.

Recommendation: option 3 if a clean upstream bundle can be assembled
for CSAPI Part 1 + Part 2; fall back to option 1 if curation pains
outweigh control benefits. Option 2 is acceptable but introduces a
build-time dependency and discipline burden that the project may not be
ready for. None of the three is unsound.

## 4. Conformance with this evaluation's predecessor history

This is the 14th issue evaluated under the rigour-first protocol. Prior
result distribution: validated 11, rejected 1 (#9 false-positive on
non-existent test framework), partially rescoped 1 (#13 — not a defect,
documentation gap). #14 is straightforwardly valid: code openly admits
unfinished state, live empirical confirms, and at least one self-bound
spec `SHALL` is violated. This is the cleanest validation of the batch
so far.

## 5. Verdict

**KEEP. Issue #14 is a real, well-framed defect with citable spec
authority.** Recommended actions (these are *suggestions to the
upstream maintainers*, not changes I will make to cs-go in this
evaluation):

- No edits to the issue text are required; the body is accurate. One
  minor refinement worth posting in the comment thread: cs-go does not
  currently declare `conf/oas30`, so the binding `SHALL` is from
  `conf/landing-page`'s `api-definition-success` (B) rather than the
  oas30 requirements class. The substance is unchanged.
- Severity P2-Important is appropriate; bumping to P1 only if formal
  certification is on the project roadmap.
- Of the three suggested fixes, option 3 (serve upstream OGC OAS
  bundle with patched `servers`) is recommended as the minimum
  acceptable resolution; option 1 if local control is preferred.

## 6. Cross-references

- Static analysis: [docs/research/evidence/issue-014/static-analysis-2026-04-30.md](../evidence/issue-014/static-analysis-2026-04-30.md)
- Spec authority: [docs/research/evidence/issue-014/spec-authority-2026-04-30.md](../evidence/issue-014/spec-authority-2026-04-30.md)
- Live test: [docs/research/evidence/issue-014/live-test-2026-04-30.md](../evidence/issue-014/live-test-2026-04-30.md)
- OAS 3.0.3: <https://spec.openapis.org/oas/v3.0.3>
- OGC API – Common Part 1: <https://docs.ogc.org/is/19-072/19-072.html>
