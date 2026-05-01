# Issue #24 — Static analysis (HEAD `39f1fd9`/`08cf161`)

Date: 2026-04-30.

## Where inline `@link` hrefs are populated — code path trace

There are **two distinct** code paths that produce an inline `@link.href`:

### 1. Server-synthesised path-derived links (the source of the bug)

`internal/api/datastream_handler.go:135-136` (POST handler):

```go
if datastream.SystemLink == nil {
    datastream.SystemLink = &common_shared.Link{Href: "systems/" + systemID}
}
```

`internal/api/control_stream_handler.go:144-145`:

```go
if cs.SystemLink == nil {
    cs.SystemLink = &common_shared.Link{Href: "systems/" + systemID}
}
```

These two assignments build a **relative** href (`"systems/<uuid>"`) and
do *not* go through `formaters.ToFunctionalAssociationHref(...)`. The
helper would absolutize the value if `associationLinksBaseURL` is
configured (and on this server it *is* configured — supplementary
`links[]` arrays come out absolute, see live evidence).

### 2. User-supplied links (already correctly absolute when caller absolutes)

The Observation/Command/SamplingFeature handlers deserialize
`procedure@link`, `result@link`, `sampledFeature@link` from the
posted body verbatim into `*common_shared.Link`. Whatever the client
posted is what gets stored. This explains the round-trip behaviour
observed live: an absolute href round-trips absolute.

## Where inline `@link` hrefs are emitted — JSON formatter

`internal/model/formaters/json_formatters/datastream_json.go`
(verified via `read_file`): the formatter copies the model verbatim
(`out := *datastream`); only the supplementary `links[]` array is
built via `formaters.ToFunctionalAssociationHref(...)` (lines 71-78).
The inline `@link` properties are emitted exactly as stored — so any
relative href in the model (from path-1 above) appears relative in the
response.

## Helper availability

`internal/model/formaters/association_links.go:371-389` defines:

```go
func ToFunctionalAssociationHref(href string) string {
    href = strings.TrimSpace(href)
    if href == "" { return href }
    if parsed, err := url.Parse(href); err == nil && parsed.IsAbs() {
        return href // already absolute → no change
    }
    if associationLinksBaseURL == "" { return href }
    if strings.HasPrefix(href, "/") { return associationLinksBaseURL + href }
    return associationLinksBaseURL + "/" + href
}
```

The helper is **idempotent**: passing an already-absolute href returns
it unchanged. This is critical — applying the helper to user-supplied
absolute hrefs is safe (no double-prefixing).

`internal/api/router.go:27` calls
`serializers.SetAssociationLinksBaseURL(cfg.API.BaseURL)` at startup,
so the base URL is configured in production and on the live test
server.

## Body's claim — verified

> **In `datastream_json.go`** … wrap the href via the same helper the
> supplementary `links[]` builder uses

This is one of two correct fix sites. The body's diagnosis identifies
the *symptom* location (formatter) but the *root cause* is upstream in
the **handlers** which stored a relative href in the first place.

## Refinement: the body's fix sketch

The body suggests applying `ToFunctionalAssociationHref(...)` in the
formatter. This is correct and sufficient (the helper is idempotent).
However, an alternative or complementary fix is to apply it at the
**handler** synthesis site, which has the advantage of normalising the
data **at write time** so what is stored matches what is read. Both
locations are defensible:

- **Formatter-only**: minimum intrusion; output is correct; storage
  retains historical (relative) hrefs.
- **Handler-only**: storage is normalised; relies on every read path
  (formatter, queries, etc.) to receive already-absolute data.
- **Both** (defence in depth): output is always correct regardless of
  storage state; new writes go in absolute too.

Recommendation: **both** sites, with a one-time backfill query optional
(not required for correctness; the formatter alone makes responses
correct).

## Affected fields — completeness check

Body lists 12 fields. Quick `git grep` in `internal/model/domains/`:

```
control_stream.go: SystemLink, ProcedureLink, DeploymentLink, FeatureOfInterest, SamplingFeatureLink (5)
datastream.go    : SystemLink, ProcedureLink, DeploymentLink, FeatureOfInterest, SamplingFeatureLink (5)
deployment.go    : Platform (Platform@link), DeployedSystems@link (1+1)
observation.go   : ProcedureLink, ResultLink (2)
command.go       : ProcedureLink (1)
sampling_feature : SampledFeatureLink (1)
system.go        : SystemKind (systemKind@link) (1)
```

Total 17 model fields × the inline JSON shape. Body lists 12 — close
but not exact. The completeness audit:

- DS: 5 ✓ (matches body)
- CS: 5 ✓ (matches body)
- Deployment: 2 ✓ (`platform@link`, `deployedSystems@link`)
- Observation: 2 ✓ (`procedure@link`, `result@link`)
- Command: 1 ✓ (`procedure@link`)
- System: 1 ✓ (`systemKind@link`)
- SamplingFeature: 1 ✓ (`sampledFeature@link`)

= 17 inline `@link` properties total. Body's count of 12 omits some
shared fields with the same name across types — re-counting using
the body's grouping:

- Datastream "five": SystemLink, ProcedureLink, DeploymentLink, FeatureOfInterest, SamplingFeatureLink — match.
- ControlStream "five": same five ✓.
- Deployment: 2 ✓
- Observation: 2 ✓
- Command: 1 ✓
- System: 1 ✓
- SamplingFeature: 1 ✓

Totals to 17, but body said "all twelve" — actually re-reading the
body it doesn't say "twelve", it lists each type separately and
totals 17 fields. Discrepancy resolved — body's enumeration is exact.

## Specific code sites that need patching for a complete fix

By type, the assignment sites that build a relative href:

- `datastream_handler.go:136` — DS create from POST `/systems/{id}/datastreams`
- `control_stream_handler.go:145` — CS create from POST `/systems/{id}/controlstreams`

For the *user-provided* link properties (`procedure@link`,
`featureOfInterest@link`, `result@link`, `sampledFeature@link`,
`systemKind@link`, `platform@link`, `deployedSystems@link`,
`deployment@link`, `samplingFeature@link`), the relative form only
arises if the **client** posts a relative path. The server should
absolutize on write *or* on read regardless.

Recommended formatter-side fix sites (per type):

- `internal/model/formaters/json_formatters/datastream_json.go`
- `internal/model/formaters/json_formatters/control_stream_json.go`
- (corresponding formatters for observation, command, deployment, system, sampling_feature — note: not all formatters exist as separate files; some encoding goes through GORM JSON marshalling directly).

The latter is a wider audit — see live test for the actual surface
that emits relative.
