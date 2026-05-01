# Issue #18 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#18 |
| Title | TimeRange.UnmarshalJSON silently drops malformed/wrong-typed time values |
| Labels | bug, priority/p2, data-integrity |
| Cited HEAD | `4b994212` |
| Verified HEAD | `08cf161` (no drift in `time_range.go` between the two) |
| **Verdict** | **KEEP** |
| Severity confirmed | P2 — data integrity |

## Summary

The issue describes a real, currently-reproducible data-integrity defect in
`internal/model/common_shared/time_range.go`. `TimeRange.UnmarshalJSON` uses
the pattern

```go
if s, ok := arr[0].(string); ok {
    if t, err := time.Parse(time.RFC3339, s); err == nil {
        tr.Start = &t
    }
}
```

in **all three** decoding branches (array, object, fallback string via
`ToTimeRange`). Both the `ok` discard (wrong-typed JSON value) and the
`err == nil` discard (malformed time string) silently produce a `TimeRange`
with `Start: nil, End: nil`. Because the field is embedded into `Datastream`
via `gorm:"embedded;embeddedPrefix:phenomenon_time_"`, the database row stores
NULL columns, and the `omitempty` JSON tag means the GET response has no
`phenomenonTime` key at all — the caller sees a 201 Created with no signal
that their input was rejected.

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-018/static-analysis-2026-04-30.md). All three branches inspected; six silent-discard sites in `UnmarshalJSON` itself, plus two more in the `ToTimeRange` fallback.
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-018/live-test-2026-04-30.md). Reproduced verbatim with `phenomenonTime:[1773100000.0,null]` (T1) and `phenomenonTime:["not-a-time","also-bad"]` (T2). Control with valid RFC 3339 strings (T3) succeeds and round-trips correctly.
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-018/spec-authority-2026-04-30.md). CSAPI Part-1 + RFC 7493 §3.4 require rejection, not silent coercion.

## Adjacent finding (not in original issue body)

The fallback `ToTimeRange(s string)` (lines 119, 124) uses

```go
t, _ := time.Parse(time.RFC3339, parts[0])
startTime = &t
```

which is **strictly worse** than the issue's described pattern: the parse
error is discarded *and* a pointer to the zero-value `time.Time`
(`0001-01-01T00:00:00Z`) is taken regardless. So a malformed string such as
`"junk/junk"` does not silently become `nil` — it silently becomes a pointer
to year-0001. This is reachable via the string-form code path in
`UnmarshalJSON` (the third branch) and via several query-param paths.
A proper fix should address this site at the same time.

## Asymmetric strictness with `Observation.ResultTime`

`internal/api/observation_handler.go:226-234` has a hand-written decoder that
returns `&decodeError{msg: "Invalid resultTime format"}` on parse failure for
`Observation.ResultTime` (scalar `time.Time`). So semantically equivalent
inputs ("a result time on a measurement-bearing resource") are validated on
Observation but silently dropped on Datastream. This corroborates that the
silent-discard pattern is a defect, not a deliberate design choice.

## Suggested fix

The fix proposed in the issue body (return `fmt.Errorf` for both the type
and parse failures in each branch) is correct as far as it goes. It should
additionally:

1. Replace the `t, _ := time.Parse(...)` sites in `ToTimeRange` with
   error-returning equivalents (or at least guard the `&t` assignment with
   `if err == nil`).
2. Consider promoting `ToTimeRange` to return `(TimeRange, error)`
   (callers in handlers can then surface a 400). This is a wider change and
   may be deferred per scope.
3. Optionally refactor `Observation.ResultTime` decoding to share the strict
   parser, eliminating the asymmetry.

## Verdict rationale

KEEP. Defect is real, currently reproducible against live HEAD, matches the
issue body byte-for-byte in both static code and live behavior. P2 severity
classification (data-integrity) is appropriate: silent acceptance of
malformed input violates the principle of least surprise and produces stored
records that misrepresent the client's intent, but does not threaten
confidentiality, authentication, or availability.
