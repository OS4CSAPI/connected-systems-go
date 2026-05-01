# Issue #20 — Static analysis (HEAD `08cf161`)

Date: 2026-04-30. Cited HEAD `4b994212`. Drift check (per #19):
`git log 4b994212..HEAD -- internal/api/observation_handler.go` returns no
commits. Code byte-identical to cited HEAD.

## Code under review — `internal/api/observation_handler.go:226-273`

```go
if resultTimeRaw, ok := raw["resultTime"].(string); ok && resultTimeRaw != "" {
    resultTime, err := time.Parse(time.RFC3339, resultTimeRaw)
    if err != nil {
        return nil, &decodeError{msg: "Invalid resultTime format"}
    }
    obs.ResultTime = resultTime
}
// ... no else branch ...

// (line 272)
if obs.ResultTime.IsZero() {
    return nil, &decodeError{msg: "resultTime is required"}
}
```

The `if resultTimeRaw, ok := raw["resultTime"].(string); ok && resultTimeRaw != ""`
gate is conjunctive on **three** conditions:

1. Key `"resultTime"` present in `raw`.
2. Value type-asserts to `string` (`ok` true).
3. String is non-empty.

When **any** of those fails, the body is skipped silently. Then the late
`obs.ResultTime.IsZero()` guard at line 272 catches the zero-value
`time.Time` and returns `"resultTime is required"`.

This single error message is therefore reached by **at least five distinct
client errors**:

| Client error | Why guard fires |
|---|---|
| Field missing | Key absent in `raw` map → first conjunct false |
| Field is JSON `null` | `nil` doesn't type-assert to `string` → `ok` false |
| Field is empty string `""` | `resultTimeRaw != ""` false |
| Field is JSON number | `float64` doesn't type-assert to `string` → `ok` false |
| Field is JSON object/array/bool | Same — type-assert fails |

Only the **malformed-string** path produces a distinct message
(`"Invalid resultTime format"`), because it's the only case that reaches the
inner `time.Parse` call.

## Field shape — confirms there's no other rescue

`internal/model/domains/observation.go:19`:

```go
ResultTime time.Time `gorm:"index;not null" json:"resultTime"`
```

Scalar `time.Time` (not `*time.Time`), so the zero-value detection at line
272 is the only signal available for "this field wasn't populated".
There's no equivalent `"phenomenonTime is required"` guard because
`PhenomenonTime` is `*time.Time` (nil-able) — see #19.

## Sibling decoder asymmetry — what this issue is about

The issue body is precise: "the conformance question is resolved by the
spec — strings only — but the error message must be honest about *why* the
request was rejected". This issue (#20) is the **UX follow-up** to #19's
data-integrity story.

#19's analysis showed that the type-assertion-failure path silently drops
both `phenomenonTime` AND `resultTime`; only the late `IsZero()` guard
incidentally surfaces the `resultTime` case. This issue (#20) addresses
the *quality* of that surfacing — the `"resultTime is required"` message
conflates ≥5 distinct error classes into one.

## Suggested fix shape (from issue body) — analysis

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

Correctly disentangles all five classes. Notes:

- Treating `null` as equivalent to "missing" is a defensible choice (RFC 7159
  permits `null` to mean "no value"); the alternative is a sixth distinct
  message `"resultTime cannot be null"`. The body's choice is reasonable.
- The proposed fix moves the early-return guard *before* the type-check,
  which means it eliminates the late `if obs.ResultTime.IsZero()` guard
  entirely (or at least obviates it for this field). That's a structural
  improvement.
- Once this is applied, the late `IsZero()` guard remains relevant only as
  a safety net for code paths that bypass `decodeObservationPayload` (e.g.
  internal callers). Should be retained as defense-in-depth.

## Conclusion

Static claims in #20 are exact:

1. ✓ Five distinct client errors all produce the message `"resultTime is required"`.
2. ✓ Only the malformed-string path produces a distinct message.
3. ✓ The wrong-type case shares its exit path with the genuinely-missing case via the late `IsZero()` guard.
4. ✓ The proposed fix shape correctly disentangles all classes.
5. ✓ Code state is byte-identical to cited HEAD `4b994212`.
