# Issue #19 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#19 |
| Title | decodeObservationPayload silently drops numeric phenomenonTime; should reject symmetrically with resultTime |
| Labels | bug, priority/p2, data-integrity |
| Cited HEAD | `4b994212` |
| Verified HEAD | `08cf161` (no drift) |
| **Verdict** | **KEEP** |
| Severity confirmed | P2 — data integrity |

## Summary

`decodeObservationPayload` in `internal/api/observation_handler.go:236-243`
gates the `phenomenonTime` decode on `if phenomenonTimeRaw, ok := raw["phenomenonTime"].(string); ok && phenomenonTimeRaw != ""`.
When the type assertion fails (e.g. client sent JSON number `1773100000`),
the entire block is skipped silently. The repository then defaults
`PhenomenonTime = &ResultTime` (line 22-25 of
`observation_repository.go`), masking the loss as a plausibly-correct
"phenomenonTime equals resultTime" GET response.

The asymmetry-with-`resultTime` framing in the issue body is precise. The
two decoder blocks are structurally identical, but `resultTime`'s silent
drop is incidentally caught by the late `obs.ResultTime.IsZero()` guard
(possible because `ResultTime` is scalar `time.Time`, not pointer), which
returns 400 with the misleading message `"resultTime is required"` (the
field WAS present, just wrong-typed). `phenomenonTime` has no equivalent
guard because it's `*time.Time` (legitimately optional).

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-019/static-analysis-2026-04-30.md). Both decoder blocks read; symmetric structure confirmed; repository default-fill confirmed; `phenomenonTime` is `*time.Time`, `resultTime` is scalar `time.Time`.
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-019/live-test-2026-04-30.md). T1 numeric `phenomenonTime: 1773100000` → 201 + GET shows `phenomenonTime == resultTime` (silent substitution). T2 numeric `resultTime` → 400 (sibling rejected). T3 malformed-string `phenomenonTime` → 400 (string-form path is strict — defect is exclusively in type-assertion path).
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-019/spec-authority-2026-04-30.md). CSAPI Part-1 §5.1 + SWE Common JSON encoding + RFC 7493 §3.4 require rejection.

## Refinement vs. issue body framing

The body says "Inconsistent with the sibling `resultTime` path, which
catches the same wrong-type input via the late `obs.ResultTime.IsZero()`
guard and returns 400". This is true but worth two clarifications:

1. The silent-drop is **symmetric in the decoder**; the asymmetry is in
   downstream visibility. Both fields' wrong-type inputs are silently
   dropped at the type-assertion step. Only the `IsZero()` not-null guard
   incidentally surfaces the `resultTime` case.
2. The 400 message `"resultTime is required"` is misleading — the field WAS
   present, it just wasn't a string. A proper fix should surface
   `"resultTime must be an RFC 3339 string"` for the wrong-type case.

This is consistent with the body's note "which simultaneously fixes the
misleading error message in the separate UX follow-up".

## Adjacent finding (not in original issue body) — `samplingFeature@id`

`internal/api/observation_handler.go:217` decodes
`samplingFeature@id` with the same `raw[...].(string); ok` pattern. A JSON
number for that field would also be silently dropped. There is no
default-fill on the repository side and the field is documented as a string
ID reference everywhere in the schema, so the wrong-type case is narrower
in practice — but the same fix shape (explicit type-check on
`raw[...]; present && v != nil`) would also harden it.

## Suggested fix — extension of body's proposal

The proposed fix in the issue body is correct as stated:

```go
if v, present := raw["phenomenonTime"]; present && v != nil {
    s, ok := v.(string)
    if !ok {
        return nil, &decodeError{msg: "phenomenonTime must be an RFC 3339 string"}
    }
    if s != "" {
        t, err := time.Parse(time.RFC3339, s)
        if err != nil {
            return nil, &decodeError{msg: "Invalid phenomenonTime format: expected RFC 3339"}
        }
        obs.PhenomenonTime = &t
    }
}
```

It should additionally:

1. Apply the same shape to `resultTime` (the body explicitly notes this).
2. Replace the late `if obs.ResultTime.IsZero() { return ... "resultTime is required" }` guard's role: it remains for the genuinely-missing case, but the wrong-type path should surface the more informative `"resultTime must be an RFC 3339 string"` first.
3. Optionally apply the same shape to `samplingFeature@id` (`raw["samplingFeature@id"]; present` → 400 if not string) for consistency.

## Verdict rationale

KEEP. Defect is real, currently reproducible against live HEAD, matches the
issue body byte-for-byte in code and behaviour. P2 (data-integrity) is
appropriate — silent acceptance with substitution is worse than silent
drop because the GET response is plausibly correct, defeating client-side
detection of the data loss.
