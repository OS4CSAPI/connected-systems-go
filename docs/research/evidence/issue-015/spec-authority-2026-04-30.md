# Issue #15 — Spec-authority evidence

**Date:** 2026-04-30
**Question:** Does the `Systems: null` property leaked into every
`/datastreams` response constitute a strict spec violation, or is it a
code-quality / convention defect?

---

## 1. Canonical schemas examined

Authority: <https://github.com/opengeospatial/ogcapi-connected-systems>
(`master` branch). The two schemas that compose the `dataStream`
response shape are:

- `api/part2/openapi/schemas/json/baseStream.json`
- `api/part2/openapi/schemas/json/dataStream.json` (which uses
  `allOf: [{ $ref: "baseStream.json" }, { properties: {…} }]`)

### `baseStream.json` defined properties (verbatim, abbreviated)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "id":          { "description": "Local resource ID …", "type": "string", "readOnly": true },
    "name":        { "description": "Human readable name …", "type": "string" },
    "description": { "description": "Human readable description …", "type": "string" },
    "validTime":   { "$ref": "../common/commonDefs.json#/$defs/TimePeriod" },
    "formats":     { "type": "array", "items": { "type": "string" }, "readOnly": true }
  },
  "required": ["id", "name", "formats"]
}
```

### `dataStream.json` adds (in its `allOf[1].properties`)

`system@link`, `outputName`, `procedure@link`, `deployment@link`,
`featureOfInterest@link`, `samplingFeature@link`, `observedProperties`,
`phenomenonTime`, `phenomenonTimeInterval`, `resultTime`,
`resultTimeInterval`, `type`, `resultType`, `live`, `schema`, `links`.

### Critical observations

- **Neither schema defines a `Systems` property** (capital or
  lower-case). The full union of legal properties is
  ⟨`id`, `name`, `description`, `validTime`, `formats`,
  `system@link`, `outputName`, `procedure@link`, `deployment@link`,
  `featureOfInterest@link`, `samplingFeature@link`,
  `observedProperties`, `phenomenonTime`, `phenomenonTimeInterval`,
  `resultTime`, `resultTimeInterval`, `type`, `resultType`, `live`,
  `schema`, `links`⟩ — twenty-one names, none of them `Systems`.
- **Neither schema declares `additionalProperties: false` or
  `unevaluatedProperties: false`.** Per JSON Schema 2020-12 default
  semantics (§10.3.2.3), the absence of `additionalProperties`
  defaults to permissive — *any* additional property is **valid by the
  schema**.

## 2. Strict-validity ruling

Because neither `dataStream.json` nor `baseStream.json` declares
`additionalProperties: false`, the leaked `"Systems": null` is **not a
strict JSON-Schema violation**. A strict schema validator
(`ajv`, `jsonschema`, `python-jsonschema`) configured with the
canonical OGC schemas will accept the cs-go response.

This is the same default-permissive semantics already established in
the issue #13 evaluation (`docs/research/issue-evaluations/issue-013.md`),
applied symmetrically: just as cs-go is *permitted* to accept unknown
properties on input, it is *permitted* to emit unknown properties on
output. Neither is a `SHALL` violation given the upstream schemas as
they currently stand.

This is exactly why the issue labels itself **P3-Minor** (`code-quality`,
`api-design`) rather than a normative-conformance bug. The categorisation
in the issue body is precisely calibrated.

## 3. Why it is still a defect worth fixing

Strict spec-permissiveness does not make the leak harmless. Three
authorities support cleaning it up:

### 3.1 RFC 7493 (I-JSON) §4.3 — interoperability recommendation

> Receivers that wish to be tolerant of malformed messages SHOULD
> attempt to ignore non-recognized members in JSON objects; senders
> that wish to be widely interoperable SHOULD only emit JSON object
> members that are defined in the protocol they implement.

The same RFC that we cited in #13 to justify *accepting* unknown
properties on input also says senders **SHOULD only emit** members
defined in the protocol. cs-go's emission of `Systems: null` is a
SHOULD-grade interoperability deviation — exactly the symmetric
counterpart of the input case.

### 3.2 Codebase convention — see static-analysis-2026-04-30.md §2

Every other internal-join/FK field in `internal/model/domains/` carries
`json:"-"`. The Datastream `Systems` field is the unique outlier in a
codebase that has otherwise applied the convention with discipline.
Resolving the inconsistency is a code-quality win.

### 3.3 Strict client behaviour

While the canonical schemas are permissive, downstream tooling often
isn't:

- OpenAPI Generator's strict-mode clients reject unknown properties.
- TypeScript `unknown`-aware clients with discriminated unions log
  warnings or fail on extras.
- CITE-style executable test suites for OGC APIs frequently configure
  `additionalProperties: false` at the test level, even when the
  source schema is permissive, to catch shape regressions.

The leak therefore raises the floor for client compatibility for no
benefit (the field is internal and always `null`).

## 4. Severity-rating sanity check

The issue labels this **P3-Minor**. Independent assessment:

| Dimension | Rating |
|---|---|
| Functional impact | None. The field is always `null`; no data loss or behavioural change. |
| Spec impact | None at the strict level (schema is permissive). SHOULD-grade RFC 7493 deviation. |
| Cross-cutting risk | Low. One field, one file, one line. |
| Fix risk | Negligible. Adding `json:"-"` cannot break any path: GORM ignores `json` tags. |
| Discoverability cost | Low — found by direct response inspection. |

**P3-Minor is exactly right.** A P2 elevation would not be defensible.

## 5. Suggested fix evaluation

The issue proposes:

```go
Systems []System `gorm:"many2many:system_datastreams;" json:"-"`
```

Independent assessment:

- ✅ Correct minimal fix.
- ✅ Preserves GORM relationship.
- ✅ Matches codebase convention exactly.
- ✅ Acceptance criteria are well-formed (struct-tag grep + JSON-key
  absence + `go build`/`go test` + live `curl | jq has("Systems")`).
- ✅ Scope-exclusion list ("don't rename, don't add lower-cased
  `systems`, don't sweep") is correct: a lower-cased `systems`
  property is not part of the canonical schema (verified §1) and
  introducing one would be a new fabrication.

The single small refinement worth noting: the issue body labels the
line as "~70" while it is actually line 57 on HEAD. Trivial; not worth
re-issuing the body for.

## Summary

The leak is **not a strict spec violation** (canonical schemas are
permissive), but it **is** a code-quality / convention / SHOULD-grade
interoperability defect. Severity P3-Minor is precisely calibrated.
The proposed one-line fix is exact and minimal.
