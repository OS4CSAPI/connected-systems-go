# Issue #21 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#21 |
| Title | Result validator ignores `nilValues` table — spec-idiomatic `"NaN"` / `"-Infinity"` strings rejected as wrong-type |
| Labels | _(none assigned at evaluation time; body declares P2 bug)_ |
| Cited HEAD | parent fork `2bbb6202` |
| Verified HEAD | local `08cf161` (validator + model files byte-identical) |
| **Verdict** | **KEEP** |
| Severity confirmed | P2 — functional gap (data integrity / spec conformance) |

## Summary

`DatastreamDataComponent.NilValues` (`internal/model/domains/datastream.go:235`)
is a fully-plumbed model field with a proper struct shape
(`{Reason string; Value json.RawMessage}` at lines 328-332), JSON tag
`nilValues`, and verified live round-trip on `GET /datastreams/{id}`. But
`validateDataComponentValue` in
`internal/api/observation_schema_validation.go` never consults it at any
of its 7 leaf-scalar branches. The spec-idiomatic carrier strings
`"NaN"` / `"Infinity"` / `"+Infinity"` / `"-Infinity"` (which the OAS31
schema explicitly enumerates as the only permitted `nilValues[].value`
forms) are rejected with `"<path> must be a number"`.

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-021/static-analysis-2026-04-30.md). Field plumbing confirmed at the model layer; `git grep -n "NilValues" -- internal/api/` returns zero matches; all 7 leaf-scalar branches of `validateDataComponentValue` reviewed and shown to be type-only.
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-021/live-test-2026-04-30.md). 4-test matrix on a datastream declaring `nilValues:[{reason:".../missing",value:"NaN"},{reason:".../BelowDetectionRange",value:"-Infinity"}]`:
  - T1 `result: {baro_altitude: "NaN"}` → 400 (spec expects 201)
  - T2 `result: {baro_altitude: "-Infinity"}` → 400 (spec expects 201)
  - T3 valid number 42.5 → 201 (control)
  - T4 `result: {baro_altitude: "banana"}` → 400 (control)
  - **T1 and T4 produce identical error messages** — proves the validator does not differentiate declared nilValue strings from arbitrary strings.
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-021/spec-authority-2026-04-30.md). Independently verified the OAS31 enum constraint (`["NaN","Infinity","+Infinity","-Infinity"]`) and the bundled working example using these exact strings. Conformance failure is unambiguous.

## Refinement vs. issue body framing

The body's framing is precise and accurate. One refinement worth recording:
the defect uniformly affects **all 7 leaf-scalar branches** (`boolean`,
`count`, `quantity`, `time`, `category`, `text`, plus the four `*Range`
branches), not just the `quantity` shown in the example. The OAS31
declares `nilValues` on every scalar component type; the validator's
absence of `nilValues` consultation is uniform across branches.

## Adjacent finding (not in original body) — sibling fields are similarly orphaned

The issue body's "cross-refs" section flags `Constraint` and `Updatable`
as audit candidates. Quick verification:

```
$ git grep -n "Constraint\|Updatable" -- internal/api/
# (no validator hits)
```

Confirmed: both `Constraint` (declared at `datastream.go` near
`NilValues`) and any `Updatable` flag round-trip on GET but are not
consulted by `validateDataComponentValue`. These are independent defects
of the same shape ("model declares it, GET round-trips it, validator
ignores it") but should be filed as separate issues — they are
out-of-scope for #21's specific `nilValues` defect.

## Suggested fix — body's proposal is correct

The body's proposed fix shape:

```go
if !isNumber(value) {
    if matchesAnyNilValue(component, value) { return nil }
    return fmt.Errorf("%s must be a number (or a declared nilValue)", path)
}
```

with `matchesAnyNilValue(component, value)` decoding each
`component.NilValues[i].Value` (`json.RawMessage`) and string-comparing —
is correct. Recommendations to the fix PR:

1. Apply uniformly to **all 7 leaf-scalar branches** (issue body lists 6;
   the `*Range` branches also need it).
2. Comparison semantics: the OAS31 enum is string-only for the value, but
   `DatastreamNilValue.Value` is `json.RawMessage`, allowing
   future-numeric values. A safe implementation should do
   `bytes.Equal(rawValue, jsonMarshalledClientValue)` to handle both
   string and (hypothetical) numeric cases consistently.
3. The body's optional v1.1 follow-on (recording the matched `reason` URI
   alongside the stored observation) is a separate enhancement; not
   needed for the conformance fix itself.
4. Add an acceptance test using the OAS31's own example body verbatim
   (see spec-authority doc).

## Verdict rationale

KEEP. Defect is real, currently reproducible against live HEAD, exactly
matches the issue body in code state, live behaviour, and spec citation.
P2 (functional gap / spec conformance) is appropriate — clients
following the OAS31 example body verbatim will receive 400 with a
misleading `"must be a number"` message. The model-layer storage is
already correct, so the fix is constrained to one validator file and
follows a well-defined shape.
