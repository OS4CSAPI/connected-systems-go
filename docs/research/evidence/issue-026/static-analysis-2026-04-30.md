# Issue #26 — Static Analysis (HEAD `5be1d42`)

## Query parameter parsing

`internal/model/query_params/query_params.go:31` defines `IDs []string` on
the common `BaseQueryParams`. Lines 56-58 parse it from the request:

```go
if ids := r.URL.Query().Get("id"); ids != "" {
    params.IDs = strings.Split(ids, ",")
}
```

**Only the `id` key is read.** Go's `url.Values.Get(...)` returns empty
string for any unknown key, so `?uid=...` and `?foo=bar` are both silently
ignored — the caller proceeds with `params.IDs == nil`.

**Verified vs. body**: exact.

## Dual semantics in the SQL filter

`git grep -n 'unique_identifier IN' -- 'internal/repository/*.go'` returns
exactly **8 hits** (one per UID-bearing repository, plus a one-off lookup
in `procedure_repository.go:146`):

| Repository | Line | SQL |
|---|---|---|
| `system_repository.go` | 393 | `id IN ? OR unique_identifier IN ?` |
| `datastream_repository.go` | 164 | same |
| `control_stream_repository.go` | 164 | same |
| `deployment_repository.go` | 122 | same |
| `procedure_repository.go` | 99 | same |
| `procedure_repository.go` | 146 | one-off `WHERE` for batch fetch |
| `property_repository.go` | 73 | same |
| `feature_repository.go` | 106 | same |
| `sampling_feature_repository.go` | 76 | same |

Both arms of the OR use the same `params.IDs` slice — so any value parsed
from `?id=` is checked against both the local UUID column and the
`unique_identifier` (URI) column. Comma-separated mixed lists work
because the slice is split before being passed.

**Verified vs. body**: body's "across all eight UID-bearing repositories"
matches exactly (System, Datastream, ControlStream, Deployment, Procedure,
Property, Feature, SamplingFeature = 8).

## Behaviour for unknown keys

`url.Values.Get(...)` is documented to "return the first value associated
with the given key" with empty string for missing keys. The cs-go
parser's `if ids := ...; ids != ""` short-circuits in that case. So
`?uid=...` is genuinely indistinguishable from `?foo=bar` from the
server's perspective — both produce an unfiltered list.

There is no aliasing, no deprecation warning, no log line. Silent
no-op.

## README state

`README.md:158-167` already includes a short Query Parameters section:

```
## Query Parameters

Common query parameters across list endpoints:

- `id` - Filter by resource ID or UID
- `q` - Full-text search
- `limit` - Page size
- `offset` - Page offset
```

**Pre-existing partial documentation**: the README **does** state that
`id` filters by "resource ID or UID". This is the body's fix in nucleus
form already, but:
- No examples are given.
- There is no explicit "do not use `?uid=`" caveat.
- A reader skimming for `uid` finds nothing (and may write the
  SensorHub-style `?uid=` workaround the body describes).

So body's premise — "there is no client-facing doc that says 'the URI
form goes into `?id=`, not `?uid=`'" — is **partly inaccurate**: the
dual-form support is documented in one terse bullet. The
discoverability gap is real but smaller than the body implies.

## Severity assessment

P4 documentation enhancement is correct. No spec compliance issue. No
behaviour change recommended in the body. The proposed fix is purely
additive prose + examples in README and OpenAPI doc landing.
