# Issue #13 — Spec authority (2026-04-30)

Sources fetched 2026-04-30 from `opengeospatial/ogcapi-connected-systems@master`.

## Datastream JSON Schema permits unknown properties

`api/part2/openapi/schemas/json/baseStream.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "id":          { "type": "string", "minLength": 1, "readOnly": true },
    "name":        { "type": "string", "minLength": 1 },
    "description": { "type": "string", "minLength": 1 },
    "validTime":   { "$ref": "../common/commonDefs.json#/$defs/TimePeriod" },
    "formats":     { "type": "array", "minItems": 1, "items": {"type": "string"}, "readOnly": true }
  },
  "required": ["id", "name", "formats"]
}
```

`api/part2/openapi/schemas/json/dataStream.json`:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "allOf": [
    { "$ref": "baseStream.json" },
    { "properties": { /* system@link, outputName, ..., schema, links */ } }
  ],
  "required": ["name", "system@link", "observedProperties",
               "phenomenonTime", "resultTime", "resultType", "live"]
}
```

**Neither schema declares `additionalProperties` or `unevaluatedProperties`.**

Per JSON Schema 2020-12 (`https://json-schema.org/draft/2020-12/json-schema-core`), in the absence of `additionalProperties` and `unevaluatedProperties`:

- `additionalProperties` is unconstrained (defaults to allow).
- `unevaluatedProperties` is unconstrained.
- An instance with extra unlisted properties is a valid instance against the schema.

Note: in `allOf` composition, even if `dataStream.json`'s second branch listed `"additionalProperties": false`, that restriction would only apply to that branch's local view — properties listed in the *referenced* `baseStream.json` would not be visible to the `additionalProperties: false` of the outer branch unless `unevaluatedProperties: false` were used. The schema authors chose neither, which is the maximally permissive option.

**Spec verdict: the canonical Datastream JSON schema explicitly permits instance documents to carry properties beyond those listed.** A server's choice to accept-and-ignore unknown properties is consistent with the schema. A server's choice to reject them is also permissible (servers may apply stricter constraints than the spec mandates), but it is not spec-required.

## OpenAPI 3.0/3.1 default

OpenAPI 3.x inherits JSON Schema's default permissive behaviour: `additionalProperties` defaults to `true` unless explicitly set to `false`. The CSAPI Part 2 OpenAPI bundle does not set `additionalProperties: false` on `dataStream` requests.

## RFC 7493 — I-JSON Message Format

Section 4.3:

> A common practice [...] is to ignore items that do not appear in any
> message specification.  This results in extreme robustness: applications
> work even with [...] items that have been added since the application
> was written.

Section 4.3 advises that an implementation receiving an unrecognised member SHOULD either ignore it or treat the message as malformed, with **ignore RECOMMENDED**. cs-go's `encoding/json` default exactly implements the recommended behaviour.

## RFC 7231 — HTTP/1.1 Semantics

§6.3.2 (HTTP 201 Created):

> The 201 (Created) status code indicates that the request has been
> fulfilled and has resulted in one or more new resources being created.

The specification places no requirement on the server to echo or persist every member of the request body. The client receives 201 Created when the resource has been created from the schema-defined fields the server understood. This is the contract the issue's reproducer hits.

## Conventions across OGC server implementations

The issue body and prior evaluations (cf. #5, #6, #7) frame SensorHub as the comparator. SensorHub's wire shape includes `uid` because SensorHub's data model includes a UID concept on multiple resource types — that is a SensorHub data-model decision, not a CSAPI Datastream-schema requirement. As established in #12's spec-authority work, the canonical Datastream schema does not declare `uid` at all. SensorHub's choice to accept and persist a `uid` on Datastream is a SensorHub-specific extension; cs-go's choice to silently ignore extension fields is JSON-convention-compliant.

I have **not** verified what pygeoapi or ldproxy do with stray fields on POST bodies; that comparison is not load-bearing because the spec is permissive.

## Summary of spec authority

| Source | Posture on unknown fields |
|---|---|
| CSAPI Part 2 `dataStream.json` | Permitted (no `additionalProperties: false`) |
| CSAPI Part 2 `baseStream.json` | Permitted (no `additionalProperties: false`) |
| JSON Schema 2020-12 default | Permitted |
| OpenAPI 3.x default | Permitted |
| RFC 7493 (I-JSON) §4.3 | Ignore recommended |
| RFC 7231 §6.3.2 (201 Created) | No echo/persist requirement |
| Postel's law | Be liberal in what you accept |
| Go `encoding/json` default | Ignore unknown members |

Every relevant authority points the same direction: **silently accepting and ignoring unknown fields is consistent with the spec, with JSON convention, and with mainstream language-default behaviour.** No authority requires either rejection or round-trip.
