# Issue #23 — Spec authority

Date: 2026-04-30. Most spec citations relevant to #23 are inherited
from #21 and #22. Only the open-schema (additional-properties) question
needs fresh spec analysis.

## Open-schema policy — what does the spec say?

The OAS31
(`docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`)
defines the SWE Common DataRecord schema descriptor. The spec
distinguishes:

1. The **schema-descriptor** (the `resultSchema` posted with a
   datastream) — defines which components are present.
2. The **observation result payload** — instances conforming to the
   schema descriptor.

For (2), SWE Common 3.0 (OGC 21-022) section 6 ("DataRecord")
specifies that a DataRecord *is* the totality of its declared `field`
list. Components not in the field list are **not part of the record**
in the SWE Common data model. The spec does not explicitly define
behaviour for "extra keys in the JSON encoding"; the JSON encoding is
defined as a faithful encoding of the SWE Common data model, so by
implication an instance with extra keys is non-conformant to the
declared schema.

That said, the spec does not mandate strict rejection. The OGC 23-002
Part 2 §9 ("Observation conformance") requires the server to
"validate the result against the result schema", but does not
prescribe whether validation is strict (closed-schema) or lax
(open-schema). Implementations have latitude.

**Conclusion**: the open-schema behaviour is a defensible
implementation choice — it is not non-conformant — but it is not
spec-mandated either, and the current behaviour of *silently*
accepting and *persisting* undeclared keys without any warning is
unusual and bears explicit documentation.

## Numeric type model — clarify SWE Common Quantity vs Count

The OAS31 (and SWE Common 3.0 §6.4) defines:

- **Quantity** — `type: number` (real-valued).
- **Count** — `type: integer`.

Both decode in Go's `encoding/json` to `float64` when the call site
uses `interface{}`/`map[string]any` without `UseNumber()`. The
validator's `isIntegerNumber` correctly enforces integer-ness via
`math.Mod(f, 1) == 0`. This is faithful to the spec's distinction.

The doc page should call this out so users understand:

- A Count component will reject `42.5` (works because
  `math.Mod(42.5, 1) != 0`).
- A Quantity component accepts `42` (because `42` decodes as `42.0`).

Both behaviours are correct; documenting them removes a class of
"why did this work / not work" questions.

## Error response shape — body's structured-error suggestion

OGC API common practice (from OGC API Features and consistent across
the ogcapi family) is the **RFC 7807 `application/problem+json`**
format:

```json
{
  "type": "https://example.org/errors/schema-validation",
  "title": "Observation result does not conform to datastream schema",
  "detail": "result.baro_altitude must be a number",
  "field": "result.baro_altitude",
  "expected": "number",
  "status": 400
}
```

This is more conventional than the body's ad-hoc `{error,field,reason,
expected}` shape and matches what other OGC API server implementations
emit. The doc deliverable should align on RFC 7807 (and a separate
issue may be warranted if the project chooses to migrate from the
current `{"error":"..."}` shape).

## Conclusion — supports the body's proposal

All five claims in the body have spec-relevant context:

1. Optional vs null — covered by #22 spec analysis (optional means
   "omitted"; null is not a permitted value).
2. nilValues — covered by #21 spec analysis (canonical absence
   carrier).
3. Open schema — neither mandated nor prohibited; documenting the
   chosen policy is the right move.
4. Numeric type model — Quantity vs Count is spec-defined; current
   implementation is faithful.
5. Error shape — RFC 7807 alignment is the strongest spec-supported
   recommendation.

The body's proposed deliverable (a single doc page) is appropriate;
the recommendations above add minor refinements to the deliverable's
content scope.
