# Issue #22 — Static analysis (HEAD `81574a7`/`08cf161` validator)

Date: 2026-04-30. File: `internal/api/observation_schema_validation.go`.

## The `datarecord` branch — verbatim

Lines 81-100:

```go
case "datarecord":
    obj, ok := value.(map[string]any)
    if !ok {
        return fmt.Errorf("%s must be an object", path)
    }
    for _, field := range component.Fields {
        if field.Name == "" {
            continue
        }
        fieldVal, exists := obj[field.Name]
        if !exists {
            if field.Optional != nil && *field.Optional {
                continue
            }
            return fmt.Errorf("%s.%s is required by datastream schema", path, field.Name)
        }
        if err := validateDataComponentValue(&field.DatastreamDataComponent, fieldVal, path+"."+field.Name); err != nil {
            return err
        }
    }
    return nil
```

## What happens for `{"baro_altitude": null, "geo_altitude": 10500}` with `baro_altitude.optional=true`

1. `value` is `map[string]any{"baro_altitude": nil, "geo_altitude": 10500.0}`
   (Go's `encoding/json` decodes JSON `null` to Go `nil` interface).
2. `fieldVal, exists := obj["baro_altitude"]` → `fieldVal = nil`, `exists = true`.
3. The `!exists` branch (which is the *only* place `Optional` is consulted) is
   skipped.
4. Recursion into `validateDataComponentValue(&field.DataComponent, nil, "result.baro_altitude")`.
5. `componentType == "quantity"`; `isNumber(nil)` returns false; emits
   `"result.baro_altitude must be a number"`.

## Vector branch (lines ~108-127) — same pattern

```go
coordVal, exists := obj[coord.Name]
if !exists {
    if coord.Optional != nil && *coord.Optional { continue }
    return fmt.Errorf("%s.%s is required by datastream vector schema", path, coord.Name)
}
```

Same shape — `Optional` is consulted only on absence.

## DataArray / Matrix branch (lines ~129-141) — N/A

Iterates array elements; `Optional` is not a concept on array elements
(SWE Common's array shape doesn't have per-element optionality), so this
branch is correctly out of scope.

## Issue body claim — verified

The body's claim that "the `datarecord` branch only checks for **field
absence** to short-circuit on `optional: true`" is **exactly correct**.
The diagnosis is precise — `Optional` short-circuit is gated solely on
the absence path.

## What the validator does **not** do

- Treat `null`-valued keys as equivalent to absence.
- Distinguish "key present with null" from "key present with wrong type"
  in the error message — both produce `"must be a number"`.

## Spec / framing considerations (recorded for the eval doc, not a static fact)

- OAS31 (`docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`): zero
  occurrences of `nullable` and zero JSON-Schema `"null"` types. Quantity
  values are `type: number`, full stop. Per JSON Schema semantics, JSON
  `null` would be schema-invalid.
- SWE Common 3.0 nilValues mechanism (#21) is the canonical way to encode
  "no value here while still recording presence"; its carrier is a
  string token (`"NaN"` etc.), not JSON null.
- Spec word for `optional`: "Specifies if the data for this component can
  be **omitted** in the datastream" (line 521 of the bundled OAS31). The
  spec says "omitted", not "omitted or null". A strict reading is that
  the *current* behaviour is spec-aligned, and the fix proposed in the
  body would relax type validation to accept a value (`null`) that no
  Connected Systems schema explicitly permits.

These spec points are documented separately in the spec-authority
evidence file; the static analysis here only confirms the body's
*description of code behaviour* is precise.
