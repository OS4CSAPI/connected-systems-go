# Issue #22 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#22 |
| Title | [P3 bug] `null` on `optional:true` numeric field is rejected, while omission is accepted — asymmetric absence |
| Verified HEAD | `08cf161` (validator file unchanged from cited fork sha) |
| **Verdict** | **KEEP — with significant reframing** |
| Severity refined | **Not P3 bug → P4 diagnostic / documentation** |

## TL;DR

The behavioural pattern the issue describes is real and reproducible.
However, **the body's framing as a bug and its proposed fix are not
correct on a strict reading of the spec**. The current server behaviour
(reject JSON `null` for an `optional: true` `Quantity`) is
spec-conformant. The proposed fix (silently accept `null` as absence)
would push the implementation toward a non-canonical encoding and
weaken schema enforcement.

The legitimate concern surviving the analysis is **diagnostic clarity**:
the error message `"must be a number"` is misleading for a `null` value,
and the asymmetry between T1 (omitted → 201) and T2 (null → 400) is
under-documented. The eval downgrades severity from "P3 bug" to "P4
diagnostic / documentation improvement".

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-022/static-analysis-2026-04-30.md). The body's claim that the `datarecord` branch only checks for absence to short-circuit `optional: true` is exactly correct: lines 91-95 of `internal/api/observation_schema_validation.go` consult `field.Optional` only inside `if !exists`. The same shape applies to the `vector` branch. JSON `null` decodes to Go `nil` interface; the key is present but value is nil; the optional check is skipped; recursion into the leaf branch (`quantity` → `isNumber(nil)` false) emits `"must be a number"`.
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-022/live-test-2026-04-30.md). 5-row matrix:
  - T1 omit optional → 201 ✓
  - T2 optional=null → 400 (asymmetric)
  - T3 valid number → 201 ✓
  - T4 omit required → 400 ("is required")
  - T5 required=null → 400 ("must be a number" — misleading; value is null, not a wrong-typed number)
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-022/spec-authority-2026-04-30.md). The OAS31 says `optional` means data "can be omitted" (line 521), not "omitted or null". Zero occurrences of `nullable` or `"null"` JSON-Schema types in the bundled OAS31 — Quantity is `number`, full stop. Per JSON Schema semantics, `null` is invalid against `type: number`. SWE Common 3.0's canonical "no value" carriers are (a) omission of an optional component, or (b) the `nilValues` mechanism (a string carrier — see #21). JSON `null` is not in the SWE Common toolkit.

## Where the body is correct

- The code-behaviour description is precise and verbatim accurate.
- The reproduction (T14/T15 in `nan-tests-head-2026-04-30.txt` referenced from #4) is exactly what the live re-run produces.
- The cross-reference to #19's "asymmetric absence" pattern is a fair pattern observation.

## Where the body is incorrect or overstated

1. **"P3 bug" framing**: The current behaviour is spec-conformant under a strict OAS31 reading. Calling it a "bug" elides that the spec defines `optional` only as "omitted", not "null", and types component values without a `null` union. P4 diagnostic/documentation improvement is a more accurate severity.

2. **"Both forms are user intent for 'no value here'"**: This is the issue author's interpretation of user intent, not a spec position. SWE Common does not treat JSON `null` as an absence carrier; it provides omission and `nilValues` (#21) for that purpose.

3. **Proposed fix is not recommended**: Silently accepting `null` as absence on `optional: true` fields:
   - Teaches clients a non-canonical encoding (drives them away from spec-correct omission and `nilValues`).
   - Weakens schema enforcement (a value that no schema permits would silently disappear).
   - May obscure real client bugs (a client setting a value to `null` due to a serializer bug would no longer get an early-warning 400).
   - Conflicts with the directional fix needed by #21 (which adds `nilValues` consultation as the canonical absence carrier; relaxing null-as-absence in parallel creates two competing absence mechanisms).

## Adjacent finding (from this evaluation, not in body)

T5 from the live matrix surfaces a *required-field=null* diagnostic
ambiguity: error message says `"must be a number"` instead of
`"is required"` (which T4's omit-required path emits). This is a
pre-existing diagnostic wart in the same code path. Since the outcome
(400) is correct, this is not a defect; it's the same diagnostic-clarity
concern as the optional-null case.

## Recommended remediation (replaces body's proposed fix)

1. **Improve diagnostics on `null`-typed leaf values** (all 7 leaf
   branches, not only `quantity`): when `value == nil` (Go) on a typed
   leaf, emit a more informative error such as:

   ```
   result.<field> is null; null is not a permitted value
   (use field omission for optional fields, or a declared nilValue
   such as "NaN" / "-Infinity" / "+Infinity" / "Infinity").
   ```

   This is purely a diagnostic improvement; it does not relax
   validation.

2. **Update API user docs** to call out the asymmetry: omission =
   "no value present"; `nilValues` carrier = "value is missing for a
   declared reason"; `null` is not accepted.

3. **Optional v1.1 follow-on** (only if there is a strong UX argument
   from real client adoption data): accept `null` as syntactic sugar
   for absence on `optional: true` fields, behind a documented
   configuration flag, **off by default** (preserves strict OAS31
   conformance for default deployments). This is *not* recommended
   without evidence of widespread client friction.

## Verdict rationale

KEEP the issue (the diagnostic concern and asymmetry are real), but
**reframe** away from "bug" toward "diagnostic / documentation
improvement", and **decline the body's proposed code fix** because
silently accepting `null` as absence is non-conformant per OAS31 and
inconsistent with SWE Common's canonical absence mechanisms (omission
and `nilValues`).

This is the case where rigorous spec analysis materially changes the
verdict: validity-first review prevents adopting a fix that would
introduce a non-conformant relaxation in the name of UX.
