# Report 01 — `/api` endpoint OpenAPI stub

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm the spec/standard sources cited below match the
>    canonical entries there. **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-01-api-endpoint-stub.md`](../upstream-issues/plan-01-api-endpoint-stub.md).
> 3. Re-read the source eval and evidence files cross-referenced in §3–§5.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#3** (`/api` endpoint stub) |
| Source fork issue | `OS4CSAPI/connected-systems-go#14` (closed not-planned, comment `4380617500`) |
| Research plan | [`../upstream-issues/plan-01-api-endpoint-stub.md`](../upstream-issues/plan-01-api-endpoint-stub.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P2** — binding spec `SHALL` violation |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD has **advanced** since the eval was written
(`2dc09f7` → `df6da0d` "code sight updates"); the relevant code path
is **unchanged**, defect still live.

```text
$ git fetch upstream
   2dc09f7..df6da0d  main -> upstream/main
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates
```

```text
$ git show upstream/main:internal/api/router.go |
    Select-String -Pattern 'getOpenAPISpec|TODO|openapi.*3\.0\.0' -Context 2,2

  r.Get("/api", func(w http.ResponseWriter, r *http.Request) {
      w.Header().Set("Content-Type",
          "application/vnd.oai.openapi+json;version=3.0")
> fmt.Fprint(w, getOpenAPISpec(cfg))
  })

> func getOpenAPISpec(cfg *config.Config) string {
>     // TODO: Implement OpenAPI 3.0 spec generation
>     return `{"openapi": "3.0.0", "info": {"title": "` + cfg.API.Title +
>         `", "version": "` + cfg.API.Version + `"}}`
  }
```

```text
$ git grep -E 'go:embed|openapi.*\.(yaml|json)' upstream/main -- internal/
(no matches)
```

```text
$ curl.exe -s -i 'https://129-80-248-53.sslip.io/csapi-go-head/api'
HTTP/1.1 200 OK
Content-Length: 88
Content-Type: application/vnd.oai.openapi+json;version=3.0
Date: Tue, 05 May 2026 17:14:03 GMT

{"openapi": "3.0.0", "info": {"title": "OGC Connected Systems API", "version": "1.0.0"}}
```

All §6 expected outcomes from the plan satisfied:
- `getOpenAPISpec` body still contains the hard-coded 2-field literal
  and the `// TODO: Implement OpenAPI 3.0 spec generation` comment.
- No `//go:embed` and no static OAS file anywhere in `internal/`.
- Live `GET /api` returns `HTTP 200`,
  `content-type: application/vnd.oai.openapi+json;version=3.0`,
  88-byte body with only `openapi` + `info` keys.

**Conclusion:** defect is live on `upstream/main` HEAD `df6da0d` and on
the live cs-go-head deployment as of 2026-05-05.

## 2. Static evidence

Source: [`../evidence/issue-014/static-analysis-2026-04-30.md`](../evidence/issue-014/static-analysis-2026-04-30.md) (refreshed in §1 above).

`internal/api/router.go` — handler and synthesis function:

```go
// OpenAPI spec
r.Get("/api", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/vnd.oai.openapi+json;version=3.0")
    fmt.Fprint(w, getOpenAPISpec(cfg))
})

func getOpenAPISpec(cfg *config.Config) string {
    // TODO: Implement OpenAPI 3.0 spec generation
    return `{"openapi": "3.0.0", "info": {"title": "` + cfg.API.Title +
        `", "version": "` + cfg.API.Version + `"}}`
}
```

`internal/api/landing_handler.go` — landing page advertises this URL as
`service-desc` with the same OAS 3.0 media type:

```go
{
    Href:  baseURL + "/api",
    Rel:   "service-desc",
    Type:  "application/vnd.oai.openapi+json;version=3.0",
    Title: "API definition",
},
```

No alternative `service-desc` link with a different media type is
offered, so content negotiation cannot route around the broken body.

The complementary endpoint `/conformance` is fully wired
(`internal/api/conformance_handler.go:26`), demonstrating the team
understands how to ship a non-stub landing-page-class endpoint —
the deficit is specific to `/api`.

## 3. Live evidence

Source: [`../evidence/issue-014/live-test-2026-04-30.md`](../evidence/issue-014/live-test-2026-04-30.md) (refreshed in §1 above).

Endpoint: `https://129-80-248-53.sslip.io/csapi-go-head/api`.

- **Status:** 200
- **Content-Type:** `application/vnd.oai.openapi+json;version=3.0`
  (the OAS 3.0 JSON media type per OGC 19-072 §12.2)
- **Content-Length:** 88 bytes
- **Body:** literally only `openapi` and `info` keys; `paths`,
  `components`, `servers`, `tags`, `security` all absent.

`/conformance` declares the binding conformance class:

```text
http://www.opengis.net/spec/ogcapi-common-1/1.0/conf/landing-page
```

It does **not** declare `…/conf/oas30`. This is a precision point
that scopes the binding `SHALL` (see §4).

## 4. Spec authority

Source: [`../evidence/issue-014/spec-authority-2026-04-30.md`](../evidence/issue-014/spec-authority-2026-04-30.md).

Every source below traces to the curated authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | Strength | Bound because… | Status |
|---|---|---|---|
| **OAS 3.0.3 §4.7.1.1** — root OpenAPI Object, `paths` REQUIRED | Hard schema rule | Response self-asserts OAS 3.0 via `Content-Type` + `"openapi":"3.0.0"` | **Violated** |
| **OGC 19-072 `/req/landing-page/api-definition-success` B** — content SHALL be an API Definition document | Strict `SHALL` | cs-go's `/conformance` declares `…/conf/landing-page` | **Violated** |
| **OGC 19-072 `/req/landing-page/api-definition-success` C** — document SHALL be consistent with the media type | Strict `SHALL` | same | **Violated** |
| **OGC 19-072 `/rec/landing-page/api-definition-oas`** — if document uses OAS 3.0, SHOULD conform to OAS 3.0 requirements class | `SHOULD` | same | Violated |

**Precision adjustment.** cs-go does **not** declare `/conf/oas30`,
so the stricter `/req/oas30/*` requirements (`oas-definition-2`,
`completeness`, `oas-impl`) are **not** directly binding. The binding
`SHALL` for this filing is `landing-page/api-definition-success`
clauses B + C alone. The OAS 3.0.3 schema rule (`paths` REQUIRED) is
binding regardless of OGC conformance declarations because the server
self-asserts OAS 3.0 in its `Content-Type`.

**Out-of-scope sources (not cited in the public extract):**
- OGC 23-001 / 23-002 (CSAPI Part 1 / Part 2) — these are the standards
  the server *implements*, not the standard the `/api` endpoint
  *violates*.
- camptocamp/ogc-client and CSAPI client-side requirements documents —
  irrelevant to a server-side `/api` defect.

## 5. Alternatives considered (internal-only)

Three fix options were evaluated; ranking and rationale follow.

### Option 1 — `//go:embed` curated OAS bundle

Embed a hand-curated OpenAPI document via `//go:embed` and serve it.

| Dimension | Assessment |
|---|---|
| Authoring delta | High — must hand-author/maintain a document covering CSAPI Part 1 + Part 2 (Features, Systems, Datastreams, Observations, Sampling Features, Procedures, plus the EDR-style query envelopes). |
| Maintenance delta | Medium — drifts from handler reality unless disciplined. |
| Spec fidelity | High control. |
| Risk | Low. One-time large authoring cost; thereafter routine. |

### Option 2 — Runtime generation via `swag` / `chi-openapi`

Annotate handlers and generate OAS at build/start time.

| Dimension | Assessment |
|---|---|
| Authoring delta | Medium — generator annotations on every handler. |
| Maintenance delta | Low *if* the team disciplines itself with tags. |
| Spec fidelity | Medium — generators tend to under-describe schemas without effort. |
| Risk | Adds a build-time dependency and discipline burden. `chi` already a dep so `chi-openapi` is plausible. |

### Option 3 — Serve upstream OGC OAS bundles with patched `servers` (recommended)

OGC publishes authoritative reusable OpenAPI building blocks at
`schemas.opengis.net/ogcapi/...`. Assemble the CSAPI Part 1 + Part 2
bundle, patch `servers` at startup, serve.

| Dimension | Assessment |
|---|---|
| Authoring delta | Low — leverages upstream-published, spec-authoritative bundles. |
| Maintenance delta | Low — track upstream bundle versioning. |
| Spec fidelity | Highest — bundle *is* the spec. |
| Risk | Media-type implication: upstream bundles are OAS 3.1, so the response `Content-Type` shifts from `…+json;version=3.0` to `…+json;version=3.1`. Non-obvious side-effect maintainer must know up front. |

**Ranking: Option 3 → Option 1 → Option 2.** Option 3 minimizes both
authoring cost and spec-fidelity risk; the OAS 3.1 media-type shift is
an upgrade, not a regression. Option 1 is the conservative fallback if
local content control is preferred. Option 2 is acceptable but trades
authoring cost for a build-time dependency and ongoing annotation
discipline.

## 6. Recommended fix

**Lead with Option 3** (serve upstream OGC OAS bundles with patched
`servers`, shift media type to `application/vnd.oai.openapi+json;version=3.1`).
**Fall back to Option 1** (`//go:embed` curated bundle) if curation
pains outweigh the control benefits.

The implementation surface is the `getOpenAPISpec(cfg *config.Config) string`
function in `internal/api/router.go` and the matching `Content-Type`
header on the `/api` route.

## 7. Scope guard

What NOT to touch as part of this filing:

- The landing page (`internal/api/landing_handler.go`) — correct as-is.
- The `/conformance` endpoint — correct as-is.
- The existing JSON encoder.
- Any code outside `internal/api/router.go`'s `getOpenAPISpec` function
  and its caller (or wherever the new bundle is wired in if Option 3
  prefers a separate file).
- The `/conformance` declarations themselves — adding `/conf/oas30`
  would trigger stricter binding and is a separate decision.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#14` was closed not-planned on the fork
  via comment `4380617500` because this is a cs-go upstream defect, not
  a fork-only concern.
- No fork-side workaround was shipped — the fork inherits the upstream
  `/api` stub verbatim.
- No maintainer triage signal exists; this is a first-time surfacing to
  upstream. Tone is empirical evidence + spec authority, no prior
  conversation references.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Lead with Option 3 alone or present all three? | **Present all three** with ranking (Option 3 → 1 → 2) and rationale. Maintainer triage signal absent; they should choose. |
| Severity framing P2 vs. P1? | **Keep P2.** Conservative; we don't know cs-go's certification ambitions. P1 reserved for the case where formal OGC certification or automated client tooling is roadmapped. |
| Mention the OAS 3.1 media-type implication of Option 3? | **Yes, explicit.** Non-obvious side-effect; surface up front so maintainer can weigh it. |
| Mention fork-side workaround in public extract? | **No.** Fork-internal context belongs in §8 above, not in §11 below. |
| Cite eval/evidence paths in the public extract? | **Yes.** Include as "full validation chain" footer links. Cheap; consistent with how the maintainer responded well to evidence-rich filings during the closure pass. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P2] GET /api returns 88-byte stub missing required 'paths' field; violates OAS 3.0.3 §4.7.1.1 and OGC 19-072 /req/landing-page/api-definition-success`

**Labels:** `bug`

---

### Context

cs-go advertises `GET /api` on the landing page as
`service-desc` with media type
`application/vnd.oai.openapi+json;version=3.0`. The current handler
returns an 88-byte string literal containing only `openapi` and `info`
fields, with the in-source comment `// TODO: Implement OpenAPI 3.0
spec generation`. This filing reports that the response body fails
both the OAS 3.0 root-object schema (`paths` is REQUIRED) and the OGC
API – Common Part 1 strict `SHALL` for the API-definition response.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

1. The response body omits the `paths` field, which OAS 3.0.3
   §4.7.1.1 marks REQUIRED at the root OpenAPI Object. The
   server's own `Content-Type` header self-asserts OAS 3.0
   conformance.
2. The response is not consistent with the API-definition contract
   declared via cs-go's own `/conformance` document
   (`…/ogcapi-common-1/1.0/conf/landing-page`), which binds the
   server to `/req/landing-page/api-definition-success` clauses
   **B** and **C**.

### Static evidence

`internal/api/router.go`:

```go
// OpenAPI spec
r.Get("/api", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/vnd.oai.openapi+json;version=3.0")
    fmt.Fprint(w, getOpenAPISpec(cfg))
})

func getOpenAPISpec(cfg *config.Config) string {
    // TODO: Implement OpenAPI 3.0 spec generation
    return `{"openapi": "3.0.0", "info": {"title": "` + cfg.API.Title +
        `", "version": "` + cfg.API.Version + `"}}`
}
```

`git grep -E 'go:embed|openapi.*\.(yaml|json)' upstream/main -- internal/`
returns zero matches — no static OAS bundle exists that the handler
could serve as an alternative.

### Live evidence

```text
$ curl -s -i https://<deployment>/api
HTTP/1.1 200 OK
Content-Type: application/vnd.oai.openapi+json;version=3.0
Content-Length: 88

{"openapi": "3.0.0", "info": {"title": "OGC Connected Systems API", "version": "1.0.0"}}
```

`GET /conformance` declares `…/ogcapi-common-1/1.0/conf/landing-page`
(binding for this filing) but does **not** declare
`…/ogcapi-common-1/1.0/conf/oas30`.

### Spec authority

- **OAS 3.0.3 §4.7.1.1** — *Fixed Fields* of the OpenAPI Object marks
  `paths` as **REQUIRED**. The server's response declares
  `"openapi": "3.0.0"` and the OAS 3.0 JSON media type, self-asserting
  OAS 3.0 conformance. (In OAS 3.1, `paths` was relaxed to
  optional-when-`webhooks`-or-`components`-is-present; that relaxation
  does not apply here.)
- **OGC 19-072** (OGC API – Common Part 1)
  `/req/landing-page/api-definition-success`:
  - **B** — *"The content of that response SHALL be an API Definition
    document."*
  - **C** — *"The API Definition document SHALL be consistent with the
    media type identified through HTTP content negotiation."*

  Both binding because cs-go's `/conformance` self-declares
  `…/ogcapi-common-1/1.0/conf/landing-page`.

**Precision note:** cs-go does not currently declare
`…/conf/oas30`, so the stricter `/req/oas30/*` requirements
(`oas-definition-2`, `completeness`, `oas-impl`) are **not** directly
binding. The binding `SHALL` for this filing is
`landing-page/api-definition-success` B + C alone, plus the OAS 3.0
schema rule itself.

### Recommended fix

Three options, presented with ranking and rationale:

1. **Serve upstream OGC OAS bundles with patched `servers`
   (recommended).** OGC publishes authoritative reusable OpenAPI
   building blocks at `schemas.opengis.net/ogcapi/...`. Assemble the
   CSAPI Part 1 + Part 2 bundle, patch `servers` at startup, serve.
   Lowest authoring delta; highest spec fidelity.
   - **Side-effect to be aware of:** upstream bundles are OAS 3.1, so
     the response `Content-Type` would shift from
     `application/vnd.oai.openapi+json;version=3.0` to
     `application/vnd.oai.openapi+json;version=3.1`, and the
     advertised media type on the landing-page `service-desc` link
     should shift in step. This is an upgrade (matches the upstream
     specs and modern tooling), but worth surfacing.
2. **`//go:embed` a curated OAS bundle** (fallback). Hand-author and
   embed an OpenAPI document covering the implemented CSAPI Part 1 +
   Part 2 surface. Highest content control; meaningful authoring cost
   but bounded.
3. **Runtime generation via `swag` / `chi-openapi`.** Annotate
   handlers and generate at build/start time. Acceptable but adds a
   build-time dependency and ongoing annotation discipline; generators
   tend to under-describe schemas without effort.

The implementation surface for any of the three is the
`getOpenAPISpec(cfg *config.Config) string` function in
`internal/api/router.go` and the matching `Content-Type` header on the
`/api` route (and the matching landing-page `service-desc` link if the
media type shifts under Option 1).

### Scope guard

Out of scope for this filing:

- The landing page handler (correct as-is).
- The `/conformance` endpoint (correct as-is).
- The `/conformance` declarations themselves — adding `/conf/oas30`
  would trigger the stricter `/req/oas30/*` binding and is a separate
  decision.
- The existing JSON encoder.

### Validation chain (links)

Internal evaluation and evidence (public on the OS4CSAPI fork):

- Synthesis report:
  `OS4CSAPI/connected-systems-go:main/docs/research/upstream-issue-reports/report-01-api-endpoint-stub.md`
- Rigour-first evaluation:
  `OS4CSAPI/connected-systems-go:main/docs/research/issue-evaluations/issue-014.md`
- Static analysis:
  `OS4CSAPI/connected-systems-go:main/docs/research/evidence/issue-014/static-analysis-2026-04-30.md`
- Live test:
  `OS4CSAPI/connected-systems-go:main/docs/research/evidence/issue-014/live-test-2026-04-30.md`
- Spec authority:
  `OS4CSAPI/connected-systems-go:main/docs/research/evidence/issue-014/spec-authority-2026-04-30.md`

Cross-reference: original fork-side issue
`OS4CSAPI/connected-systems-go#14` (closed not-planned because this is
a cs-go upstream defect, not a fork-only concern).

---

## 11. Filing record

- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/1>
- **Filed:** 2026-05-05
- **Filed by:** OS4CSAPI fork research stream (via MCP GitHub integration)
- **Audit verdict:** `pass` (see [`../report-audit-log.md`](../report-audit-log.md#report-01-api-endpoint-stub))

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
