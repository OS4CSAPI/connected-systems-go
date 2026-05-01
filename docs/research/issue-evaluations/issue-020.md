# Issue #20 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#20 |
| Title | Misleading error 'resultTime is required' when field is present but wrong type |
| Labels | enhancement, priority/p3, ux |
| Cited HEAD | `4b994212` |
| Verified HEAD | `08cf161` (no drift; verified during #19) |
| **Verdict** | **KEEP** |
| Severity confirmed | P3 — UX |

## Summary

`decodeObservationPayload` in `internal/api/observation_handler.go:226-273`
gates the `resultTime` decode on the conjunctive
`if resultTimeRaw, ok := raw["resultTime"].(string); ok && resultTimeRaw != ""`.
When **any** of the three conjuncts fails (key absent, value not a string,
string empty), the block is skipped silently. The late
`if obs.ResultTime.IsZero()` guard at line 272 then catches the zero-value
`time.Time` and returns `"resultTime is required"` — a message that is
correct for one of the cases but misleading for at least four others.

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-020/static-analysis-2026-04-30.md). Decoder block + late guard read; field declared as scalar `time.Time` (not pointer) which is what makes the `IsZero()` rescue possible.
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-020/live-test-2026-04-30.md). 7-row matrix:
  - T1 numeric, T3 missing, T4 null, T5 empty string, T6 object, T7 bool → all return `"resultTime is required"` (6 distinct client errors → 1 message).
  - T2 malformed-string-but-still-string `"2026-04-30"` → `"Invalid resultTime format"` (only path with a distinct accurate message).
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-020/spec-authority-2026-04-30.md). No strict spec violation (CSAPI doesn't prescribe error wording), but defeats the spirit of RFC 7807 §3 "identify the problem".

## Refinement vs. issue body framing

The body identifies the wrong-type-vs-missing conflation (T1 vs. T3). Live
testing widens the conflation to **6 distinct client errors → 1 message**
(adding T4 null, T5 empty string, T6 object, T7 bool). The issue body's
proposed fix correctly disentangles all six because it branches on
`!present || v == nil` first, then `v.(string); !ok`, then `s == ""` —
each producing a distinct message.

One nuance: the body's fix groups `null` with "missing" under
`"resultTime is required"`. RFC 8259 §3 distinguishes the two, but
collapsing them is defensible because the client correction is the same
("provide a non-null string value"). Worth noting in the fix PR; not a
defect.

## Adjacent finding (not in original body) — `phenomenonTime` parallel

The same conflation problem applies in mirror to `phenomenonTime`, but
inverted: because `obs.PhenomenonTime` is `*time.Time` (nullable, line 18
of `observation.go`), there is **no** late `IsZero()` guard. So
`phenomenonTime: 1773100000` is silently dropped with no error at all —
that's the data-integrity story in #19. Once #19's fix shape lands
(explicit type-check before parse, with distinct messages), the
`phenomenonTime` decoder will produce the same disentangled error messages
that #20 requires for `resultTime`. So #19 and #20 fixes naturally fold
into a single decoder change.

## Suggested fix — body's proposal is correct

The body's proposed code is correct as stated:

```go
v, present := raw["resultTime"]
if !present || v == nil {
    return nil, &decodeError{msg: "resultTime is required"}
}
s, ok := v.(string)
if !ok {
    return nil, &decodeError{msg: "resultTime must be an RFC 3339 date-time string"}
}
if s == "" {
    return nil, &decodeError{msg: "resultTime must be a non-empty RFC 3339 date-time string"}
}
t, err := time.Parse(time.RFC3339, s)
if err != nil {
    return nil, &decodeError{msg: "Invalid resultTime format: expected RFC 3339"}
}
obs.ResultTime = t
```

Recommendations to the fix PR:

1. Apply the same shape to `phenomenonTime` (paired with #19 as the body
   notes).
2. Retain the late `if obs.ResultTime.IsZero()` guard as defense-in-depth
   for code paths that bypass `decodeObservationPayload` (e.g. internal
   callers).
3. Optionally apply to `samplingFeature@id` (per #19's adjacent finding) for
   consistency.

## Verdict rationale

KEEP. UX defect is real, currently reproducible against live HEAD, and the
live-test matrix shows the conflation is wider than the body claims (6 →
1, not just 2 → 1). P3 classification (UX, not data-integrity, not
security) is appropriate; the data-integrity twin is #19. Fix is low-risk
(error-message branching only) and naturally folds into the #19 PR.
