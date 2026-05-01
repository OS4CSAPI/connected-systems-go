# Issue #18 — Static analysis (HEAD `08cf161`)

Date: 2026-04-30. Issue cites HEAD `4b994212`. Verified
`git log 4b994212..HEAD -- internal/model/common_shared/time_range.go internal/model/domains/datastream.go internal/api/datastream_handler.go internal/api/observation_handler.go`
returns no commits — code state is byte-identical to the cited HEAD.

## Code under review — `internal/model/common_shared/time_range.go`

The function quoted in the issue body is exact. Full
`UnmarshalJSON` body, lines 47-100:

```go
func (tr *TimeRange) UnmarshalJSON(b []byte) error {
    if len(b) == 0 || string(b) == "null" {
        *tr = TimeRange{}
        return nil
    }

    // try array form
    var arr []interface{}
    if err := json.Unmarshal(b, &arr); err == nil {
        if len(arr) > 0 && arr[0] != nil {
            if s, ok := arr[0].(string); ok {                       // (A1) silent type-assert
                if t, err := time.Parse(time.RFC3339, s); err == nil {  // (A2) silent parse
                    tr.Start = &t
                }
            }
        }
        if len(arr) > 1 && arr[1] != nil {
            if s, ok := arr[1].(string); ok {                       // (A3) silent
                if t, err := time.Parse(time.RFC3339, s); err == nil {  // (A4) silent
                    tr.End = &t
                }
            }
        }
        return nil
    }

    // try object form
    var obj struct {
        Start *string `json:"start,omitempty"`
        End   *string `json:"end,omitempty"`
    }
    if err := json.Unmarshal(b, &obj); err == nil {
        if obj.Start != nil && *obj.Start != "" {
            if t, err := time.Parse(time.RFC3339, *obj.Start); err == nil {  // (O1) silent
                tr.Start = &t
            }
        }
        if obj.End != nil && *obj.End != "" {
            if t, err := time.Parse(time.RFC3339, *obj.End); err == nil {    // (O2) silent
                tr.End = &t
            }
        }
        return nil
    }

    // try string form
    var s string
    if err := json.Unmarshal(b, &s); err == nil {
        *tr = ToTimeRange(s)                                         // ToTimeRange also discards parse errors via `t, _ := time.Parse(...)`
        return nil
    }

    return fmt.Errorf("unsupported TimeRange JSON format")
}
```

**Six** silent-discard sites in `UnmarshalJSON` itself (A1-A4, O1, O2), plus
two more in the `ToTimeRange` fallback (line 119 and 124 — `t, _ := time.Parse(time.RFC3339, parts[0])`
and identical for `parts[1]`). The issue body's claim "in **every** branch"
is exact.

The `ToTimeRangeFromSlice` helper at line 137 is *also* silent (`if t, err := time.Parse(...); err == nil { startTime = &t }`),
but that function is reached only from query-param parsing, not from
`UnmarshalJSON` — separate concern.

## Datastream field shape — confirms storage path

`internal/model/domains/datastream.go:34-35`:

```go
PhenomenonTime         *common_shared.TimeRange `gorm:"embedded;embeddedPrefix:phenomenon_time_" json:"phenomenonTime,omitempty"`
PhenomenonTimeInterval *string                  `gorm:"type:varchar(64)" json:"phenomenonTimeInterval,omitempty"`
```

`gorm:"embedded;embeddedPrefix:phenomenon_time_"` means `Start *time.Time` →
column `phenomenon_time_start`, `End *time.Time` → column `phenomenon_time_end`.
When `UnmarshalJSON` silently produces `TimeRange{Start: nil, End: nil}`, GORM
writes both columns as NULL — exactly the "stored as empty/nil" claim in the
issue body. The `json:"phenomenonTime,omitempty"` tag combined with the custom
`MarshalJSON` (which returns `[null]` for an all-nil range) means the GET
response either omits the field entirely (because `tr` is itself a nil
pointer) or returns `[null]` if the embed produced a non-nil-but-empty
`TimeRange` — matching the live behavior observed.

`ResultTime` at line 36 has the same shape and same storage semantics.

## Asymmetric strictness — confirmed

`internal/api/observation_handler.go:226-234`:

```go
if resultTimeRaw, ok := raw["resultTime"].(string); ok && resultTimeRaw != "" {
    resultTime, err := time.Parse(time.RFC3339, resultTimeRaw)
    if err != nil {
        return nil, &decodeError{msg: "Invalid resultTime format"}
    }
    obs.ResultTime = resultTime
}
```

Observation `resultTime` decoding does NOT use `TimeRange.UnmarshalJSON`. It
parses out of a `map[string]interface{} raw` and explicitly returns
`&decodeError` on parse failure — `Observation.ResultTime` is `time.Time`
(scalar), not `*TimeRange`. So the asymmetry is **real** but worth being
precise about: it is asymmetry between `Observation.ResultTime` (scalar
`time.Time`, hand-written strict decoder) and `Datastream.PhenomenonTime/ResultTime`
(`*TimeRange`, generic silent decoder). Same conceptual data ("a result time
on an observable"), different code paths, different strictness.

## Marshal-side (read-back path) note

`MarshalJSON` at lines 19-39 emits `[start, end]` or `[start]` with each
element either an RFC3339 string or `nil`. When both are nil, the result is
the JSON literal `[null]`. Combined with `*common_shared.TimeRange` being a
pointer field with `omitempty`, the post-roundtrip shape is "no
`phenomenonTime` key at all" — also matching live behavior.

## Conclusion

Static claims in #18 are exact:

1. ✓ Six silent-discard sites in `UnmarshalJSON` (4 in array form, 2 in object form).
2. ✓ Plus 2 more in the string-form fallback via `ToTimeRange` (`t, _ := time.Parse(...)`).
3. ✓ `Datastream.PhenomenonTime` is `*common_shared.TimeRange` with `gorm:"embedded;embeddedPrefix:phenomenon_time_"`.
4. ✓ Asymmetric strictness vs. `Observation.ResultTime` (scalar time, hand-written strict decoder).
5. ✓ Code state is byte-identical to the cited HEAD `4b994212`.
