# OGC 23-002 (CSAPI Part 2) + SWE Common — NaN / nilValue extracts

Source: `.tmp-csapi-part2.yaml` (bundled OAS31 from `OGC/ogcapi-connectedsystems`).

## §9.7 Observation result encoding — Quantity component nilValues

Multiple example fixtures in the spec (around lines 1867, 2058, 2093, 2123) declare `nilValues` exactly as follows for a numeric Quantity field:

```yaml
- name: temp
  type: Quantity
  definition: http://mmisw.org/ont/cf/parameter/air_temperature
  label: Room Temperature
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

The example `value:` payloads in the spec are `NaN`, `'-Infinity'`, `+Infinity`. In JSON encoding (RFC 8259 forbids these as bare numeric literals) the spec-idiomatic carrier is the **string form** (`"NaN"`, `"-Infinity"`, `"+Infinity"`) — the same form OSH SensorHub accepts and the same form `OSHConnect-Python` / `json.dumps()` emit when given a Python `float('nan')`.

## Implication for issue #4

1. **The string `"NaN"` is the spec idiom** for representing missing/sentinel numeric values when the publisher uses JSON encoding. Rejecting it with "must be a number" is **non-conformant** with how the spec writers expect these values to flow.
2. **The `nilValues` collection is a publisher-declared per-field table** mapping reason URIs → sentinel encodings. A spec-conformant validator for a Quantity field must:
   - Accept any `value` that is a number, **OR**
   - Accept any value (number or string) whose JSON serialization equals one of the declared `nilValues[*].value` for that field.
3. Without this, a publisher cannot use the spec's nilValue mechanism end-to-end against cs-go.

## SWE Common 3.0 — nilValues definition

The SWE Common Data Model encoding rules define `swe:nilValues` as a list of `(reason, value)` pairs attached to scalar components. The reason URIs come from an OGC code list (e.g. `http://www.opengis.net/def/nil/OGC/0/{missing,inapplicable,withheld,unknown,BelowDetectionRange,AboveDetectionRange,…}`). The carrier value is encoded per the data stream's encoding rule — for JSON, the per-publisher convention is the string form for non-finite floats since RFC 8259 has no native NaN/Infinity.

## RFC 8259 (JSON) — relevant clauses

- §6 Numbers: "Numeric values that cannot be represented in the grammar below (such as Infinity and NaN) are not permitted."

So a literal `NaN` / `Infinity` in the JSON wire format is invalid JSON and must be rejected. This is correct behaviour and is what cs-go does for T8/T9 (returns 400 with `invalid character 'N'`). This is **separate** from the `"NaN"` string question.
