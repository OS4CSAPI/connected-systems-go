# Issue #21 — Static analysis (HEAD `08cf161`)

Date: 2026-04-30. Cited HEAD: `2bbb6202` (parent fork). Verified against
local HEAD `08cf161`; the validator file
`internal/api/observation_schema_validation.go` and the model field
`internal/model/domains/datastream.go:235` are byte-identical to the
referenced state.

## Model field — `NilValues` IS plumbed

`internal/model/domains/datastream.go:235`:

```go
NilValues  []DatastreamNilValue  `json:"nilValues,omitempty"`
```

`internal/model/domains/datastream.go:328-332`:

```go
// DatastreamNilValue maps reserved nil value entries.
type DatastreamNilValue struct {
    Reason string          `json:"reason"`
    Value  json.RawMessage `json:"value"`
}
```

Plumbing complete: field defined, JSON tag set, struct shape matches the
spec convention `(reason, value)`. Round-trip on GET confirmed live (see
`live-test`).

## Validator — `NilValues` is NEVER consulted

`internal/api/observation_schema_validation.go`, leaf-scalar branches:

```go
case "boolean":
    if _, ok := value.(bool); !ok {
        return fmt.Errorf("%s must be a boolean", path)
    }
    return nil

case "count":
    if !isIntegerNumber(value) {
        return fmt.Errorf("%s must be an integer", path)
    }
    return nil

case "quantity":
    if !isNumber(value) {
        return fmt.Errorf("%s must be a number", path)
    }
    return nil

case "time", "category", "text":
    if _, ok := value.(string); !ok {
        return fmt.Errorf("%s must be a string", path)
    }
    return nil

case "countrange", "quantityrange", "timerange", "categoryrange":
    arr, ok := value.([]any)
    if !ok || len(arr) != 2 {
        return fmt.Errorf("%s must be a 2-item array", path)
    }
    return nil
```

None of these branches reads `component.NilValues`. `git grep -n "NilValues" -- internal/api/`
returns **zero matches**:

```
internal/model/common_shared/characteristics.go:102:    NilValues  json.RawMessage ...
internal/model/common_shared/characteristics.go:134:            NilValues  json.RawMessage ...
internal/model/common_shared/characteristics.go:149:    c.NilValues = aux.NilValues
internal/model/domains/datastream.go:235:       NilValues  []DatastreamNilValue ...
internal/model/generators/generators_common_shared.go:126:      // set UOM/Constraint/NilValues for richer payloads
internal/model/generators/generators_common_shared.go:132:      cw.NilValues = nilv
```

All hits are model-layer / generators. Zero validator hits. The half-built
feature claim is exact.

## Conformance — affects all 7 leaf-scalar branches

The OAS31 schema (verified independently — see `spec-authority`) declares
`nilValues` on **all** scalar component types: `Quantity`, `Count`, `Time`,
`Category`, `Text`, `Boolean`, plus their `*Range` variants. The current
validator's missing-consultation defect therefore affects **all** of them
uniformly. The issue body's enumeration ("`quantity`, `count`, `time`,
`category`, `text`, `boolean`") matches.

## Code state confirmation

- `git log --oneline -- internal/api/observation_schema_validation.go internal/model/domains/datastream.go` shows last commit on these files as `1562201 update datastreams` (predates the cited HEAD by some history; both files in current HEAD have not changed since).
- The validator-leaf-branch code shown above is verbatim what's currently shipping.
