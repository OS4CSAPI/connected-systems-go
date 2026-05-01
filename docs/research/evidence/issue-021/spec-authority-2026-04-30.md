# Issue #21 — Spec authority

Date: 2026-04-30.

## Spec citation in issue body — independently verified

The issue cites OGC API — Connected Systems Part 2 OAS31 (file
`.tmp-csapi-part2.yaml` line 2058) for the canonical example using
`"NaN"` / `"-Infinity"` strings. I verified against my local copy of the
bundled OAS31:

`docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`

### 1. The `nilValues` value field is constrained to a string enum

Inside the bundled SWE Common subschema for scalar components (referenced
by `Quantity`, `Count`, `Time`, `Category`, `Text`, `Boolean`):

```yaml
nilValues:
  description: Defines reserved values with special meaning (e.g., missing, out-of-range, etc.)
  ...
    type: string
    enum:
      - NaN
      - Infinity
      - +Infinity
      - '-Infinity'
```

The OAS31 schema explicitly enumerates the four spec-idiomatic carrier
strings as the *only* permitted values for the `value` member of
`nilValues[]`. This is unambiguous — clients are **required** to use these
exact string forms (because RFC 8259 forbids bare `NaN`/`Infinity` JSON
numeric literals; the spec authors chose strings as the carriers).

### 2. The OAS31 ships a working example with these exact strings

Same file, in the request-body example for an Observation-bearing
endpoint:

```yaml
uom:
  code: Cel
nilValues:
  - reason: http://www.opengis.net/def/nil/OGC/0/missing
    value: NaN
  - reason: http://www.opengis.net/def/nil/OGC/0/BelowDetectionRange
    value: '-Infinity'
  - reason: http://www.opengis.net/def/nil/OGC/0/AboveDetectionRange
    value: +Infinity
```

This is the exact shape the issue body's reproduction uses (same
`reason` URIs, same `value` strings).

## SWE Common Data Model 3.0

The OAS31 references SWE Common 3.0 (OGC 21-022) for the `nilValues`
semantics. SWE Common §6.4 ("Soft typed scalars") defines `nilValues` as
"a list of reserved values that have special meanings […] NaN, Infinity,
−Infinity, sentinel codes, etc." A conforming server MUST interpret a
result-value matching any declared `nilValues[i].value` as the
corresponding `reason`, not as a type violation.

## Conformance verdict

The current cs-go behaviour — rejecting `"NaN"` with a type error — is
**non-conformant** with:

1. The OAS31 schema enum constraint (the value `"NaN"` is explicitly
   permitted as a `nilValues[].value`, and a result matching it MUST be
   accepted as that field's value).
2. The bundled OAS31 working example (which would 400 against the current
   implementation).
3. SWE Common Data Model 3.0 §6.4 nilValues semantics.

## Severity

P2 (functional gap) is appropriate. This is not a parsing or input-type
question — the spec is unambiguous. The model layer fully implements
storage; only the validator-side consumption is missing. A future client
or interoperability test that POSTs the OAS31 example body verbatim will
fail against cs-go.

## Acceptance test (suggestion to fix PR)

The simplest acceptance test is the OAS31's own example: declare a
Quantity field with `nilValues: [{reason: ".../missing", value: "NaN"}]`,
POST an observation with `result: {field: "NaN"}`, expect 201. Should
pass after the validator fix.
