# `GET /api` returns 86-byte stub with no paths or schemas

> **Filing status:** Filed as [#14](https://github.com/OS4CSAPI/connected-systems-go/issues/14) on 2026-04-30. Drafted from issue #1 evaluation §2.8 T2.

| Field | Value |
|---|---|
| **Severity** | P2-Important |
| **Category** | API Design / Documentation / Spec Conformance |
| **Source** | Live empirical testing during issue #1 evaluation (parallel HEAD deploy) |
| **Ownership** | connected-systems-go (server) |
| **Affects** | cs-go HEAD `4b99421` (and likely every commit) |
| **Labels** | `bug`, `api-design`, `spec-conformance`, `discovery-mode` |

---

## Goal

`GET /api` must return a complete, conformant OGC API service description
(OpenAPI 3.x with full `paths`, `components`, etc.) — not the current 86-byte
placeholder — so that machine clients can discover the service contract per
OGC API – Common requirements.

## Problem Statement

The route `GET /api` is currently wired to a hardcoded string literal stub
that contains only `openapi`, `info.title`, and `info.version`. There are no
`paths`, no `components`, no `tags`, no `servers`. The handler is even labelled
with a `// TODO` comment marking it as unimplemented.

**Affected code:**

```go
// internal/api/router.go lines 255-267 (HEAD 4b99421)
// OpenAPI spec
r.Get("/api", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/vnd.oai.openapi+json;version=3.0")
    fmt.Fprint(w, getOpenAPISpec(cfg))
})

// ...

func getOpenAPISpec(cfg *config.Config) string {
    // TODO: Implement OpenAPI 3.0 spec generation
    return `{"openapi": "3.0.0", "info": {"title": "` + cfg.API.Title + `", "version": "` + cfg.API.Version + `"}}`
}
```

The landing page at `GET /` advertises this URL as `service-desc`
(`internal/api/landing_handler.go` line 42, with `Type:
"application/vnd.oai.openapi+json;version=3.0"`), so consumers are pointed at
this stub by spec-conformant link traversal.

**Reproduction (from issue #1 §2.8 T2):**

```bash
curl -s "https://129-80-248-53.sslip.io/csapi-go-head/api"
# {"openapi": "3.0.0", "info": {"title": "OGC Connected Systems API", "version": "1.0.0"}}
#
# 86 bytes. No paths. No components. No servers.
```

**Impact:**

- **Spec non-conformance.** OGC API – Common requires the
  `service-desc` link target to actually describe the service. Returning an
  empty stub fails Conformance Class "OpenAPI 3.0".
- **Tooling brokenness.** Generators (Swagger UI, Redoc, openapi-generator,
  ogc-client) that follow the `service-desc` link receive zero useful
  information and either render an empty page or fail to start.
- **Discoverability loss.** Connected Systems is large and evolving; an empty
  spec means clients must hand-craft requests against documentation external
  to the service, exactly the failure mode OGC API was designed to avoid.

## Suggested fix (one of)

The cs-go maintainer holds decision authority. Possible directions:

1. **Embed a static OAS file generated from the bundled OAS31 schema.** The
   repo already references the OGC schema; generate `api/openapi.yaml` once,
   embed via `//go:embed`, serve with content-negotiation between JSON/YAML.
   Predictable; one source of truth; no runtime cost.
2. **Generate at runtime from chi route metadata + struct reflection.** Use
   a library (e.g. `github.com/swaggo/swag` or `chi-openapi`) to introspect
   handlers and JSON tags. More work, but stays in sync with code drift
   automatically.
3. **Return the upstream OGC OAS31 schema verbatim with `servers` patched in.**
   Lowest delta from official spec; trivial to implement; gives clients a
   correct contract immediately. Risk: server reports schemas it does not
   actually implement.

## Files to modify

| File | Action | Rough size | Purpose |
|---|---|---|---|
| `internal/api/router.go` | Modify (`getOpenAPISpec`) | ~5–30 lines | Replace stub with real spec source |
| `api/openapi.yaml` (or similar) | Add (option 1) | many lines | Static spec bundle |
| `internal/api/router_test.go` | Add | ~30–60 lines | Smoke-test the response contains `paths` and `components` |

## Scope — what NOT to touch

- ❌ Do **not** also rework the landing page (`/`) link section; it already
  advertises the right `service-desc` URL.
- ❌ Do **not** add new business endpoints under this issue. Scope is the
  `/api` response body and its tests only.
- ❌ Do **not** change the response media type (`application/vnd.oai.openapi+json;version=3.0`).
- ❌ Do **not** also fix the missing `/conformance` payload if it has the
  same problem; file a separate issue if so.

## Acceptance criteria

- [ ] `GET /api` returns a JSON body whose top-level object contains a
      non-empty `paths` map.
- [ ] The response body is at least 4 KB (sanity threshold; real specs are
      tens of KB).
- [ ] The response object contains a `components.schemas` map with at least
      one entry.
- [ ] Response continues to be served as
      `application/vnd.oai.openapi+json;version=3.0`.
- [ ] `go build ./cmd/server` exits 0.
- [ ] `go test ./internal/api/...` exits 0.

### Acceptance gate (verification commands)

```powershell
go build ./cmd/server
go test ./internal/api/...
# Then, against a running server:
$body = Invoke-RestMethod -Uri "http://localhost:8080/api"
if (-not $body.paths) { throw "no paths" }
if (-not $body.components.schemas) { throw "no components.schemas" }
"$body" | Measure-Object -Character
# expected: more than 4096 characters
```

**Expected output:** all commands exit 0; the assertions pass; character
count exceeds 4096.

## Definition of done / closing workflow

1. Tick every acceptance-criteria checkbox.
2. Push the implementing commit to `OS4CSAPI/connected-systems-go` and copy
   its SHA.
3. Close with a summary comment containing: the commit SHA, files modified,
   the acceptance-gate command output (paths count, components count, body
   size), and which suggested-fix option was chosen.

## Dependencies

- **Blocked by:** none.
- **Blocks:** any client-side tooling that wants to auto-generate from the
  service description (e.g. ogc-client integration tests for Connected
  Systems).
- **Related:**
  - Issue #1 — surfaced this stub during live empirical testing.
  - The bundled OGC OAS31 schemas in the repo's `schemas/` (or equivalent)
    directory are a candidate source for the static spec.

## References

| # | Document | What it provides |
|---|---|---|
| 1 | [`internal/api/router.go`](../../../internal/api/router.go) lines 255–267 | The stub handler and TODO marker |
| 2 | [`internal/api/landing_handler.go`](../../../internal/api/landing_handler.go) line 42 | Where `service-desc` is advertised |
| 3 | [`docs/research/issue-evaluations/issue-001.md`](../issue-evaluations/issue-001.md) §2.8 T2 | Reproduction context |
| 4 | OGC API – Common Part 1, Requirement Class "OpenAPI 3.0" | Spec authority for service-desc completeness |
| 5 | OGC API – Connected Systems Part 1 / Part 2 bundled OAS31 schemas | Candidate spec content |

## Filing checklist

- [ ] Severity reflects "spec non-conformance + tooling brokenness" (P2 chosen)
- [ ] Acceptance gate is 100% automated; no manual UI or eyeballing
- [ ] Files-to-modify table matches the chosen-option options
- [ ] Definition of Done section is intact
- [ ] Dependencies block names #1 and the OGC schema bundle
- [ ] References table includes spec citation, evidence, and affected code
- [ ] Scope fence forbids touching landing-page links and `/conformance`
- [ ] Title under ~120 characters and follows
      `<verb-or-state> <object> <observed behaviour>` form
