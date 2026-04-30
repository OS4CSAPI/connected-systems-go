# Issue #3 — Time-field encoding leniency: numeric epoch vs RFC 3339 string

- **Issue**: [OS4CSAPI/connected-systems-go#3](https://github.com/OS4CSAPI/connected-systems-go/issues/3) — *Research / spike: should observation `resultTime`/`phenomenonTime` accept numeric epoch in addition to ISO 8601 strings?*
- **Type**: Research spike — but live verification on HEAD surfaced two material bugs the issue body did not mention.
- **Test surface**: HEAD deployment `https://129-80-248-53.sslip.io/csapi-go-head/` (commit `4b994212`), production deployment `https://129-80-248-53.sslip.io/csapi-go/` (pre-`1562201`).
- **Evidence directory**: `docs/research/evidence/issue-003/`
- **Date**: 2026-04-30

---

## 1. Load-bearing claims (from issue body)

| ID | Claim | Verdict |
|----|-------|---------|
| C1 | Posting an Observation with numeric `resultTime` (e.g. `1773100000.0`) returns HTTP 400. | **Confirmed.** |
| C2 | OSH SensorHub accepts both numeric epoch and RFC 3339 strings. | **Plausible, not re-tested here**; resolved by spec analysis below — SensorHub's leniency is non-conformant convenience. |
| C3 | Publisher clients (e.g. OSHConnect-Python) currently coerce numeric → string before POST. | **Plausible, not re-tested**; aligns with spec-required behaviour. |
| C4 | The cs-go server-side error message is unhelpful. | **Confirmed.** Body is `{"error":"resultTime is required"}`, even when the field IS present (as a number). The misleading wording is the *real* user-facing defect in this code path. |

---

## 2. Verification

### 2.1 Spec analysis — OGC 23-002 (CSAPI Part 2)

`.tmp-csapi-part2.yaml` (bundled OAS31), Observation schema:

```yaml
phenomenonTime:
  type: string
  format: date-time
resultTime:
  type: string
  format: date-time
required:
  - id
  - datastream@id
  - resultTime
```

JSON Schema `type: string` + `format: date-time` ⇒ **RFC 3339 strings only**. Numeric epoch is not a conformant encoding for these fields.

**Therefore: cs-go's rejection is correct per spec.** The SWG choice would be (a) keep strict, fix UX & sibling silent-failure; or (b) add a documented non-conformant-but-pragmatic numeric extension. Recommendation in §6.

Evidence: [docs/research/evidence/issue-003/spec-extract-2026-04-30.md](../evidence/issue-003/spec-extract-2026-04-30.md)

### 2.2 Empirical reproduction (HEAD)

Full transcript: [docs/research/evidence/issue-003/time-tests-head-2026-04-30.txt](../evidence/issue-003/time-tests-head-2026-04-30.txt)

| # | Input `resultTime` | HTTP | Body | Notes |
|---|--------------------|------|------|-------|
| T1 | `"2026-04-30T12:00:00Z"` (canonical RFC 3339) | **201** | (empty) | Baseline — works. |
| T2 | `1773100000.0` (numeric float) | **400** | `{"error":"resultTime is required"}` | Misleading — field *is* present, just wrong type. |
| T3 | `1773100000` (numeric int) | **400** | `{"error":"resultTime is required"}` | Same. |
| T4 | `1773100000000` (epoch ms) | **400** | `{"error":"resultTime is required"}` | Same. |
| T5 | `"2026-04-30"` (date only) | **400** | `{"error":"Invalid resultTime format"}` | Correct, distinguishable error. |
| T6 | `""` (empty string) | **400** | `{"error":"resultTime is required"}` | OK. |
| T7 | `null` | **400** | `{"error":"resultTime is required"}` | OK. |
| T8 | (omitted) | **400** | `{"error":"resultTime is required"}` | OK. |
| T9 | `resultTime` valid string, `phenomenonTime: 1773100000` (numeric) | **201** | (created) | **Hidden bug:** numeric `phenomenonTime` is **silently dropped** without warning. |
| T10 | `"2026-04-30T12:00:00.123Z"` (RFC 3339 + ms) | **201** | | Accepted — Go `time.RFC3339` admits fractional seconds. |
| T11 | `"2026-04-30T12:00:00-05:00"` (TZ offset) | **201** | | Accepted. |
| T12 | Datastream POST, `phenomenonTime: [1773100000.0, null]`, `resultTime: [1773100000.0, null]` | **201** | (created) | **Hidden bug, severe:** datastream is created but stored with NO `phenomenonTime`/`resultTime` at all (GET response omits both fields). Silent data loss. |
| T13 | Datastream POST, same shape but string elements | **201** | | Accepted. |

### 2.3 Source attestation — `decodeObservationPayload`

`internal/api/observation_handler.go` (HEAD `4b994212`):

```go
if resultTimeRaw, ok := raw["resultTime"].(string); ok && resultTimeRaw != "" {
    resultTime, err := time.Parse(time.RFC3339, resultTimeRaw)
    if err != nil {
        return nil, &decodeError{msg: "Invalid resultTime format"}
    }
    obs.ResultTime = resultTime
}
// … identical pattern for phenomenonTime …
if obs.ResultTime.IsZero() {
    return nil, &decodeError{msg: "resultTime is required"}
}
```

The type assertion `raw["resultTime"].(string)` **silently fails** when the JSON value is a number. Control falls through, `obs.ResultTime` stays zero, and the final guard reports the field as "required" — even though the client clearly sent it. This is the mechanism behind T2/T3/T4. There is no `else` branch for the wrong-type case.

### 2.4 Source attestation — silent-swallow in `TimeRange.UnmarshalJSON`

`internal/model/common_shared/time_range.go`:

```go
if s, ok := arr[0].(string); ok {
    if t, err := time.Parse(time.RFC3339, s); err == nil {
        tr.Start = &t
    }
}
```

Two layers of silent failure: the type assertion drops non-strings, and the inner parse drops malformed strings. Neither returns an error. `Datastream.PhenomenonTime` and `Datastream.ResultTime` are `*common_shared.TimeRange`, so a numeric or malformed time array on Datastream POST results in a successfully-stored record with empty time fields. This is the mechanism behind T12.

### 2.5 Production cross-check

Production deployment uses the same `decodeObservationPayload` (verified by `sed`-extracting the source on the prod VM). Numeric `resultTime` returns the same misleading 400 body. Confirms the defect is not HEAD-only.

---

## 3. Reproduction summary

```bash
# Numeric resultTime — strict reject with misleading message
curl -X POST $BASE/datastreams/$DS/observations \
  -H "Content-Type: application/json" \
  -d '{"resultTime":1773100000.0,"result":42}'
# → HTTP 400 {"error":"resultTime is required"}

# Numeric phenomenonTime — silently dropped
curl -X POST $BASE/datastreams/$DS/observations \
  -H "Content-Type: application/json" \
  -d '{"resultTime":"2026-04-30T12:00:00Z","phenomenonTime":1773100000,"result":42}'
# → HTTP 201, observation stored (phenomenonTime absent / defaulted)

# Datastream with numeric TimeRange arrays — silently dropped
curl -X POST $BASE/systems/$SYS/datastreams \
  -H "Content-Type: application/json" \
  -d '{"name":"DSnum",...,"phenomenonTime":[1773100000.0,null],"resultTime":[1773100000.0,null],...}'
# → HTTP 201, datastream stored without any phenomenonTime/resultTime
```

---

## 4. Verdicts

| Surface | Behaviour | Spec-conformant? | Defect class |
|---------|-----------|------------------|--------------|
| Observation `resultTime`, numeric input | Reject 400 | **Yes** (rejection is correct) | UX (misleading message) |
| Observation `phenomenonTime`, numeric input | Silently dropped, 201 | **No** | **P2 — silent data loss / inconsistent strictness** |
| Datastream `TimeRange` (`phenomenonTime`/`resultTime`), numeric or malformed | Silently dropped, 201, no error | **No** | **P2 — silent data loss** |
| Observation `resultTime`, string but bad format | Reject 400 with clear message | Yes | None |
| Observation, valid RFC 3339 with ms or TZ offset | Accept 201 | Yes | None |

---

## 5. Reasoning

The issue body framed this as a single conformance/leniency question. Live testing reveals it is **three distinct concerns conflated**:

1. **Conformance**: Numeric epoch is not allowed by the spec. Strict rejection is correct. (No code change required if SWG keeps strict.)
2. **Error UX**: When the client sends `resultTime` as a number, the server says the field is missing. This is wrong on its face and produces the exact developer-confusion that prompted this issue. **Trivial fix**, high leverage.
3. **Silent failure asymmetry**: The same wrong-type input that is *rejected* on `Observation.resultTime` is *silently swallowed* on `Observation.phenomenonTime` and on `Datastream.{phenomenonTime,resultTime}`. Asymmetric strictness across sibling fields is a worse defect than either pure-strict or pure-lenient — it produces apparently-successful POSTs that store empty time fields. **Real bugs**, must fix regardless of the leniency decision.

---

## 6. Recommendation

Adopt **strict-with-clear-errors** rather than introducing leniency:

1. **Fix the misleading message.** In `decodeObservationPayload`, branch the type-assertion failure path explicitly:
   ```go
   if v, present := raw["resultTime"]; present {
       s, ok := v.(string)
       if !ok || s == "" {
           return nil, &decodeError{msg: "resultTime must be a non-empty RFC 3339 date-time string"}
       }
       t, err := time.Parse(time.RFC3339, s)
       if err != nil {
           return nil, &decodeError{msg: "Invalid resultTime format: expected RFC 3339"}
       }
       obs.ResultTime = t
   }
   ```
   Same change for `phenomenonTime` — closes the silent-drop on T9.

2. **Fix `TimeRange.UnmarshalJSON`.** Return an error from each branch on (a) wrong element type and (b) RFC 3339 parse failure. Closes T12 silent data loss on Datastream POST.

3. **Document the encoding** in `docs/api/observations.md` with an explicit note that numeric epoch is not accepted (spec-mandated), and provide a one-line conversion recipe for the common publisher languages (Python, Go, JS).

4. **Defer leniency.** Numeric-epoch acceptance is a non-conformant convenience extension. If the SWG later wants it, gate it behind a config flag and emit a deprecation header on lenient acceptance — but the current issue does not require it.

---

## 7. Open questions

- Does the SWG want to entertain numeric leniency as a profile/extension? (Out of scope for this evaluation; spec-authoritative answer is "strings only".)
- Should `phenomenonTime` default to `resultTime` when the client sends an unparseable value, or always reject? Spec says it *defaults to the same value as resultTime when absent* — semantically that means absent ≠ wrong-type; wrong-type should reject.

---

## 8. Follow-up surfaces

| # | Type | Title | Priority | Status |
|---|------|-------|----------|--------|
| F1 | Bug | `TimeRange.UnmarshalJSON` silently swallows non-string array elements and parse failures — Datastream POST with numeric/malformed times silently stores empty TimeRange | P2 (data integrity) | filed below |
| F2 | Bug | `decodeObservationPayload` silently drops `phenomenonTime` when it is sent as a non-string (e.g. numeric epoch) — should reject symmetrically with `resultTime` | P2 (data integrity) | filed below |
| F3 | Enhancement | Misleading error `"resultTime is required"` when the field is present but wrong type — should say `"resultTime must be an RFC 3339 string"` | P3 (UX) | filed below |

---

## Changelog

- 2026-04-30 — Initial evaluation. Static analysis + 13 live test cases on HEAD + production cross-check + OGC 23-002 OAS31 schema review. Findings F1/F2/F3 to be filed as separate issues.
