# Issue #25 — Static Analysis (HEAD `5f079c3`)

## Link struct supports the fields

`internal/model/common_shared/links.go:35-41`:

```go
type Link struct {
    Href  string  `json:"href"`
    Rel   string  `json:"rel,omitempty"`
    Type  string  `json:"type,omitempty"`
    Title string  `json:"title,omitempty"`
    UID   *string `json:"uid,omitempty"`
}
```

JSON-marshal correctly omits empty optionals via `omitempty`. UID is a pointer
so a literal empty string can be distinguished from absence.

**Verified vs. body**: exact match.

**Spec coverage**: spec Link object (OAS31 line 312-372) supports
`href` (required), `rel`, `type`, `hreflang`, `title`, `uid`, `rt`, `if`. The
struct covers `href`, `rel`, `type`, `title`, `uid` — i.e. the enumerated
optionals minus `hreflang`/`rt`/`if`. For the proposed enrichment the
struct's coverage is sufficient; no schema changes required.

## Where inline `@link` is populated

`git grep -n "SystemLink\|ProcedureLink\|SamplingFeatureLink"` returned the
following synthesis sites (all build a Link with `Href` only):

| File:line | Context |
|---|---|
| `internal/api/datastream_handler.go:136` | `&common_shared.Link{Href: "systems/" + systemID}` (POST datastream) |
| `internal/api/datastream_handler.go:173` | falls back to `existing.SystemLink` on update — preserves whatever was stored |
| `internal/api/control_stream_handler.go:145` | identical shape for ControlStream |
| `internal/api/control_stream_handler.go:183` | update fallback |
| `internal/repository/datastream_repository.go:102` | `datastream.ProcedureLink = &common_shared.Link{Href: "procedures/" + kindID}` (kind-derived) |
| `internal/repository/control_stream_repository.go:102` | identical for ControlStream |

In every server-synthesized path, only `Href` is set. `Rel`/`Type`/`Title`/`UID`
remain at zero values and are omitted by `omitempty` on serialization.

**Verified vs. body**: body's claim that "only `Href` is populated" is exact for
the inline `@link` properties. The body specifically calls out
`appendDatastreamAssociationLinks` — that helper actually builds the
**supplementary `links[]` array**, not the inline `@link` field; but the
broader thrust (server-synthesized links lack the optional metadata) holds
for both arms (see below).

## Supplementary `links[]` builder

`internal/model/formaters/json_formatters/datastream_json.go:63-83`:

```go
func appendDatastreamAssociationLinks(ds *domains.Datastream) common_shared.Links {
    links := append(common_shared.Links{}, ds.Links...)
    if ds.ID == "" { return links }
    observationLink := common_shared.Link{
        Rel:  common_shared.OGCRel("observations"),
        Href: formaters.ToFunctionalAssociationHref("/datastreams/" + ds.ID + "/observations"),
    }
    systemLink := common_shared.Link{
        Rel:  common_shared.OGCRel("systems"),
        Href: formaters.ToFunctionalAssociationHref("/systems/" + *ds.SystemID),
    }
    links = append(links, systemLink)
    return append(links, observationLink)
}
```

Sets `Rel` and `Href` only; `Type`/`Title`/`UID` not populated. `Type` could
trivially be `application/geo+json` (system) or
`application/json;profile=...` (observations collection). `Title` requires a
secondary fetch (system `name`); `UID` likewise (system `uid`). So this site
is **partially** addressable without a database round-trip (`Type` constants
only) and **fully** addressable with a join.

## Round-trip preservation

`internal/repository/datastream_repository.go:226-239` reads back
`SystemLink`/`ProcedureLink`/`SamplingFeatureLink` from JSONB columns and
calls `GetId(...)` to extract the foreign key — the full Link object is
preserved verbatim in storage (`gorm:"type:jsonb"` on the model fields).
Thus client-supplied optional fields persist (verified live in
`live-test-2026-04-30.md` T2).

## Severity assessment

P4 quality-of-life enhancement is correct. No conformance violation
(optional fields are optional). Pure additive enrichment with zero risk to
existing clients.
