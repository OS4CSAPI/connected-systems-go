# Issue #19 — Static analysis (HEAD `08cf161`)

Date: 2026-04-30. Cited HEAD `4b994212`. Drift check
`git log 4b994212..HEAD -- internal/api/observation_handler.go` returns
no commits — code state is byte-identical to the cited HEAD.

## Code under review — `internal/api/observation_handler.go:226-243`

```go
if resultTimeRaw, ok := raw["resultTime"].(string); ok && resultTimeRaw != "" {
    resultTime, err := time.Parse(time.RFC3339, resultTimeRaw)
    if err != nil {
        return nil, &decodeError{msg: "Invalid resultTime format"}
    }
    obs.ResultTime = resultTime
}

if phenomenonTimeRaw, ok := raw["phenomenonTime"].(string); ok && phenomenonTimeRaw != "" {
    phenomenonTime, err := time.Parse(time.RFC3339, phenomenonTimeRaw)
    if err != nil {
        return nil, &decodeError{msg: "Invalid phenomenonTime format"}
    }
    obs.PhenomenonTime = &phenomenonTime
}
```

Both blocks gate on the type-assertion `raw[...].(string); ok`; if `ok` is
false (e.g. the JSON value was a number), the entire block is skipped with
no error. This is **structurally symmetric** — the two branches share the
identical decode shape. The asymmetry the issue body identifies is
downstream:

```go
if obs.ResultTime.IsZero() {
    return nil, &decodeError{msg: "resultTime is required"}
}
```

`obs.ResultTime` is `time.Time` (scalar, line 19 of
`internal/model/domains/observation.go` — `ResultTime time.Time gorm:"index;not null" json:"resultTime"`).
A skipped decode leaves it at the zero value, which the late
`IsZero()` guard catches. There is **no equivalent guard** for
`obs.PhenomenonTime` because that field is `*time.Time` (line 18 — pointer,
optional) and a skipped decode legitimately leaves it nil.

So the asymmetry is real and is structural, but the framing is worth being
precise about: the silent-drop is **symmetric** (both fields' wrong-type
inputs are silently discarded by the decoder), and only the *visibility* is
asymmetric (the `IsZero()` not-null guard incidentally exposes the
`resultTime` case).

## Default-fill that masks the silent drop — `internal/repository/observation_repository.go:21-34`

```go
func (r *ObservationRepository) Create(observation *domains.Observation) error {
    if observation.PhenomenonTime == nil {
        t := observation.ResultTime
        observation.PhenomenonTime = &t
    }
    if observation.ResultTime.IsZero() {
        now := time.Now().UTC()
        observation.ResultTime = now
        if observation.PhenomenonTime == nil {
            observation.PhenomenonTime = &now
        }
    }
    return r.db.Create(observation).Error
}
```

When the decoder silently drops `phenomenonTime`, the repository sets
`observation.PhenomenonTime = &observation.ResultTime` — exactly the
"defaulted to resultTime" behaviour observed live. The client sees a 201
Created and a GET response showing `phenomenonTime == resultTime`, with no
indication that the original numeric input was discarded.

## Sibling field shapes — confirms it's a 1-of-1 case

`grep` against `decodeObservationPayload` shows the only other
`raw[...].(string); ok` decode is `samplingFeature@id` (line 217). That field
is plain optional with no late guard, so a numeric value would also be
silently dropped — but `samplingFeature@id` is documented as "string ID
reference" everywhere in the schema, so the wrong-type-input class is
narrower (and there's no domain-default-fill that would mask the loss).

## Conclusion

Static claims in #19 are exact:

1. ✓ `phenomenonTime` decode skips silently on type-assertion failure.
2. ✓ `resultTime` decode is structurally identical but caught downstream by `obs.ResultTime.IsZero()` → 400 `"resultTime is required"`.
3. ✓ The late `IsZero()` guard is a *symptom* catching the silent drop, not a real type validator — the message is misleading (the field WAS present, it just wasn't a string).
4. ✓ Repository auto-fills `PhenomenonTime = &ResultTime` on nil, masking the loss as "defaulted" rather than "dropped".
5. ✓ Code state is byte-identical to cited HEAD `4b994212`.
