# Issue #23 — Static analysis + claim verification (HEAD `c8b7704`/`08cf161`)

Date: 2026-04-30. File: `internal/api/observation_schema_validation.go`.

## Verifying each of the 5 claims in the issue body

### Claim 1 — `optional: true` covers absence but not `null`

**Verified TRUE.** This is the central finding of #22. See
`docs/research/evidence/issue-022/static-analysis-2026-04-30.md` for the
full code-path trace. Body's cross-ref to #22 is exact.

### Claim 2 — `nilValues` declared/persisted/returned but not consumed by validator

**Verified TRUE.** This is the central finding of #21. See
`docs/research/evidence/issue-021/static-analysis-2026-04-30.md`.
`git grep -n "NilValues" -- internal/api/` returns zero matches; round-trip
GET preserves the array. Body's cross-ref to #21 is exact.

### Claim 3 — Open-schema policy: extra fields silently accepted and persisted

**Verified TRUE — re-tested live on HEAD `08cf161`.**

The `datarecord` branch (lines 81-100) iterates *only* over
`component.Fields` (declared schema fields). It never iterates over
keys present in the input `obj` that are not in the declared schema.
Therefore any undeclared key in the JSON object is silently skipped by
validation and passes through to persistence verbatim.

Live test (DS `c838cd58-ac38-40a7-b966-ec93b4a9b7d5`):

```
POST /datastreams/$ds/observations
{"result":{"geo_altitude":10500,
           "undeclared_extra_field":"surprise",
           "another_typo_field":42}}
→ 201; obsId=d21a2f6a-699d-4e19-8b6c-f47ff157e742

GET /observations/d21a2f6a... → result:
  {"geo_altitude":10500,
   "another_typo_field":42,
   "undeclared_extra_field":"surprise"}
```

Both undeclared fields round-trip verbatim. A typo in a declared field
name is therefore indistinguishable from an extension at the validation
layer (the typo'd field passes, the original-named field is treated as
absent — which surfaces as 400 only if the original was non-optional;
otherwise the typo silently wins).

### Claim 4 — `isNumber` accepts `float64` only; integers decode as `float64`

**Verified TRUE.** Lines 341-352:

```go
func isNumber(v any) bool {
    _, ok := v.(float64)
    return ok
}

func isIntegerNumber(v any) bool {
    f, ok := v.(float64)
    if !ok { return false }
    return math.Mod(f, 1) == 0
}
```

This works in practice because Go's `encoding/json` decodes every JSON
number into `interface{} = float64` by default (no `UseNumber()`
configured at the call site). Refinement to body's claim: this is fine
for typical use, but note (a) `isIntegerNumber` will accept `42.0`
written as a float in JSON — which matches JSON's spec since JSON has
no integer/float distinction at the syntax level, and (b) a future
switch to `json.Decoder.UseNumber()` would silently break both
predicates by handing `json.Number` (a string) to the leaf branches.
Worth documenting as a stability concern.

### Claim 5 — Error format requires regex parsing for field path

**Verified TRUE.** Across all leaf branches the format is the
positional dotted path concatenated with a free-form English message:
`result.<field> must be a number`, `result.<field> is required by
datastream schema`, `result.<field>.<sub> must be ...`, etc. There is
no machine-readable field separator; clients must regex-parse to
recover the dotted path. The body's suggestion of a structured error
shape is a legitimate improvement.

## Summary of static + live verification

All 5 enumerated claims in the body are factually correct on HEAD
`08cf161`. None of these behaviours are documented anywhere in the
repo:

```
$ git ls-files docs/api/    →    (does not exist)
```

`git grep -l -i "nilValues|optional.*omit|extra.*field" -- docs/ README.md`
returns only our own research evidence files (issue-004/005/021/022
work) — there is no user-facing doc page describing any of these
behaviours.

## What this issue is asking for

Pure documentation work — no code changes proposed. The body asks for
a single reference page (`docs/api/observation-result-schema.md`) plus
two minor follow-on decisions: (a) whether to tighten or formalise the
open-schema policy, and (b) the structured error-response shape.

## Validity verdict

The issue's claims are all verified. The dependency-ordering note
("depends on #21 and #22 landing first") is exactly correct given the
reframing of #22 — the doc should reflect the post-#21 world (nilValues
as canonical absence carrier) and the post-#22 reframed world (null
rejection is intentional and spec-conformant; document the omission /
nilValues alternatives clearly).
