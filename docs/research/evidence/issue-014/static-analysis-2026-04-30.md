# Issue #14 — Static analysis evidence

**Date:** 2026-04-30
**Repo HEAD evaluated:** `629e2c404243a2a34223492da2b5a14a6597b284`
**Scope:** `GET /api` handler, landing page advertisement, presence of any
embedded OAS asset.

---

## 1. The `/api` handler is a hard-coded literal stub

`internal/api/router.go` (lines 255–267):

```go
// OpenAPI spec
r.Get("/api", func(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "application/vnd.oai.openapi+json;version=3.0")
    fmt.Fprint(w, getOpenAPISpec(cfg))
})
```

```go
func getOpenAPISpec(cfg *config.Config) string {
    // TODO: Implement OpenAPI 3.0 spec generation
    return `{"openapi": "3.0.0", "info": {"title": "` + cfg.API.Title +
        `", "version": "` + cfg.API.Version + `"}}`
}
```

Observations:

- The handler returns a string literal interpolated only with
  `cfg.API.Title` and `cfg.API.Version`.
- The body has only two top-level keys: `openapi` and `info`.
- It contains **no** `paths`, `components`, `servers`, `tags`, or `security`
  members.
- The author marked the function with `// TODO: Implement OpenAPI 3.0 spec
  generation`, openly signalling the implementation is unfinished.
- The `Content-Type` is set to the OGC-recommended OAS 3.0 JSON media type
  (`application/vnd.oai.openapi+json;version=3.0` — see OGC 19-072 §12.2).

## 2. The landing page advertises `/api` as `service-desc`

`internal/api/landing_handler.go` (lines 41–51):

```go
{
    Href:  baseURL + "/api",
    Rel:   "service-desc",
    Type:  "application/vnd.oai.openapi+json;version=3.0",
    Title: "API definition",
},
{
    Href:  baseURL + "/conformance",
    Rel:   "http://www.opengis.net/def/rel/ogc/1.0/conformance",
    Type:  "application/json",
    Title: "Conformance declaration",
},
```

Observations:

- `/api` is the canonical machine-readable API definition link advertised
  to clients per OGC API – Common Part 1 §9.2 (`service-desc` relation).
- The advertised media type matches the response's `Content-Type` header,
  so a client cannot avoid the broken body via content negotiation —
  it is the only representation offered.

## 3. No static OAS bundle exists in the repository

Searched the entire workspace for an embedded OpenAPI document:

- `grep -RIn "//go:embed"` → **0 matches**
- `grep -RIn "embed.FS"` → **0 matches**
- `find . -name "openapi.yaml" -o -name "openapi.json" -o -name "swagger.yaml" -o -name "swagger.json"` → **0 matches**
- No `schemas/`, `api/openapi.*`, or equivalent directory exists.

There is no static OAS asset bundled with the binary that the handler
could serve. The 88-byte string is the only OpenAPI document the server
is capable of producing on this code path.

## 4. The complementary endpoint `/conformance` is fully wired

For comparison, the analogous OGC API – Common landing-page sibling
endpoint is correctly implemented:

- `internal/api/router.go:81` —
  `r.Get("/conformance", conformanceHandler.GetConformance)`.
- `internal/api/conformance_handler.go:26` defines
  `GetConformance` returning a JSON object with a populated `conformsTo`
  array.

This shows the team understands how to ship a non-stub
landing-page-class endpoint; the `/api` deficit is specific to the
OpenAPI-document handler.

## 5. The cs-go conformance declaration claims `/conf/landing-page`

Live `GET /conformance` (see live-test-2026-04-30.md) declares, among
others:

- `http://www.opengis.net/spec/ogcapi-common-1/1.0/conf/core`
- `http://www.opengis.net/spec/ogcapi-common-1/1.0/conf/landing-page`
- `http://www.opengis.net/spec/ogcapi-common-1/1.0/conf/json`
- `http://www.opengis.net/spec/ogcapi-connectedsystems-1/1.0/conf/api-common`
- `http://www.opengis.net/spec/ogcapi-connectedsystems-2/1.0/conf/api-common`

It does **not** declare `/conf/oas30` of OGC API – Common Part 1. This
nuance matters when stating which spec requirements bind cs-go (see
spec-authority-2026-04-30.md): the `landing-page` requirements bind
strictly; the `oas30` requirements are not asserted, but the media type
header on the response still implicitly invokes the OAS 3.0
specification itself.

## Summary

The repository openly contains an admitted-incomplete stub for the
machine-readable API definition. The landing page advertises this stub
as `service-desc` with an OAS 3.0 media type. No static OAS bundle
exists that the handler could serve as an alternative. The empirical
premise of issue #14 is fully supported by the static code and
repository state.
