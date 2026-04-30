# Issue #13 — Static analysis (2026-04-30)

Repo @ `c6ca4c6` (`origin/main`).

## 1. Deserialize path is end-to-end standard `encoding/json`

`internal/api/datastream_handler.go::CreateDatastream` (line 125) and `UpdateDatastream` (line 163) both invoke:

```go
datastream, err := h.fc.Deserialize(contentType, r.Body)
```

`h.fc` is `*formaters.MultiFormatFormatterCollection[*domains.Datastream]`. Its `Deserialize` (`internal/model/formaters/formatter.go:164`):

```go
func (m *MultiFormatFormatterCollection[Domain]) Deserialize(contentType string, reader io.Reader) (Domain, error) {
    formatter := m.GetFormatter(contentType)
    return formatter.Deserialize(context.Background(), reader)
}
```

`GetFormatter("application/json")` returns the `DatastreamJSONFormatter` registered in `router.go::buildDatastreamFormatterCollection`. Its `Deserialize` (`internal/model/formaters/json_formatters/datastream_json.go`):

```go
func (f *DatastreamJSONFormatter) Deserialize(ctx context.Context, reader io.Reader) (*domains.Datastream, error) {
    var datastream domains.Datastream
    if err := json.NewDecoder(reader).Decode(&datastream); err != nil {
        return nil, err
    }
    return &datastream, nil
}
```

Standard Go `encoding/json` default behaviour: `Decoder.Decode` ignores JSON object keys that have no matching struct field. `DisallowUnknownFields()` is **not** called anywhere in the package — `grep -RnE 'DisallowUnknownFields' internal/` returns zero matches.

Conclusion: the issue's empirical premise (top-level `uid` is silently dropped) is real and is the standard Go default, applied uniformly to every unknown JSON field, not specific to `uid`.

## 2. Datastream struct has no catch-all field for arbitrary unknown JSON

`grep` for any field that could absorb arbitrary unknown keys on `Datastream`:

```
internal/model/domains/datastream.go:
  - Formats, SystemLink, ProcedureLink, DeploymentLink,
    FeatureOfInterest, SamplingFeatureLink, ObservedProperties,
    Schema, Links — all gorm:"type:jsonb" but each has a typed Go type
    (Link, []DatastreamObservedProperty, etc.)
  - No top-level `Properties common_shared.Properties` field
  - No top-level `json.RawMessage` member
  - No `map[string]interface{}` catch-all
```

Other domain types (`DatastreamObservedProperty`, `DatastreamSchema`, etc.) carry `Extensions common_shared.Properties` for a *bounded* extension surface, but none of those is reachable from a top-level `uid` or `properties` key in the request body — a top-level `properties` key in the POST body has no struct mapping at all and is silently dropped wholesale.

This means the silent-drop is **complete and uniform** for top-level unknown keys. There is no partial round-trip, no hidden `properties` jsonb echo, no warning header. The behaviour matches what Go's `encoding/json` does by default when given a JSON object with extra members.

## 3. Spec authority — `additionalProperties` is unset

Canonical CSAPI Part 2 schemas at `opengeospatial/ogcapi-connected-systems@master`:

- `api/part2/openapi/schemas/json/baseStream.json` — `type: object`, `properties: {id, name, description, validTime, formats}`, `required: [id, name, formats]`. **No `additionalProperties` declaration.**
- `api/part2/openapi/schemas/json/dataStream.json` — `allOf: [baseStream.json, { properties: {...} }]`, `required: [name, system@link, observedProperties, phenomenonTime, resultTime, resultType, live]`. **No `additionalProperties` declaration on either branch.** **No `unevaluatedProperties` declaration.**

Per JSON Schema 2020-12 default semantics, the absence of `additionalProperties: false` (and `unevaluatedProperties: false`) means **unknown properties are permitted by the schema**. The canonical CSAPI schema explicitly leaves the door open for instance documents to carry properties beyond those listed.

This is the dispositive spec finding for #13: **the schema permits unknown properties in the request body.** A server that silently ignores them is consistent with the schema. A server that rejected them with HTTP 400 would be applying a stricter interpretation than the spec mandates — which is allowed (servers can be strict beyond the spec) but is a deployer choice, not a defect fix.

## 4. JSON / REST convention

- **RFC 7493 (I-JSON), §4.3:** "an I-JSON receiver MUST [...] either ignore the values [it does not understand] or treat the message as malformed [...] ignore is RECOMMENDED."
- **RFC 7231 §6.3.2 (HTTP 201 Created):** "the request has been fulfilled and has resulted in one or more new resources being created." No requirement that the server echo or persist every field of the request body.
- **OpenAPI 3.x default:** `additionalProperties` defaults to `true`; `additionalProperties: false` is opt-in.
- **Postel's law (Robustness Principle):** "Be conservative in what you do, be liberal in what you accept."

The standard convention across HTTP/JSON APIs is to ignore unknown fields. Deviations exist (gRPC over JSON, some strict-mode servers), but they are explicit opt-ins, not defaults. Go's `encoding/json` defaults exactly to this convention.

## 5. The three options the issue proposes

The issue body proposes three remediation options. Reviewing them against the spec + convention:

1. **`DisallowUnknownFields` globally on Datastream deserializer.** Returns HTTP 400 on any unknown key. **Breaking change** for any client that legitimately sends extension fields, vendor-specific properties, or forward-compat fields. Goes against the spec's permissive `additionalProperties` posture and against I-JSON convention. The acceptance-criteria gate of "existing GET responses unchanged for records created without `uid`" is silent on the breakage this would cause for clients sending other unknown fields (e.g. `extensions`, `metadata`, vendor namespaces).
2. **Reject only `uid` specifically.** Singles out one key with no spec basis for asymmetric treatment. Brittle — would need updates whenever the spec evolves the field set. Implements special-case logic to enforce a "should not be sent" signal that the spec does not encode.
3. **Re-add a no-op `uid` field with `gorm:"-"`.** Resurrects a vestigial field for the sole purpose of round-tripping it through the response. Adds technical debt; locks in a wire shape the spec doesn't define.

All three options are spec-conformant in a permissive sense (the spec doesn't *forbid* strict parsing or echo), but none of them is spec-mandated. Each is a deployer-side UX decision, not a defect fix.

## 6. Generic vs uid-specific behaviour

The issue's framing centres on the `uid` field, but the underlying behaviour is uniform: **every** unknown top-level key is silently dropped, not just `uid`. Live testing (see live-test evidence) confirms that `foo`, `properties`, and `uid` all disappear identically. Singling out `uid` (Option 2) would create asymmetric semantics with no spec grounding.

This generic-not-specific characterisation is important for the verdict: there is no `uid`-specific defect on cs-go HEAD; there is only the standard JSON-decoder default applied generically.
