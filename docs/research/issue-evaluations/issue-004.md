# Issue #4 — Numeric NaN / nilValue handling for observation result fields

- **Issue**: [OS4CSAPI/connected-systems-go#4](https://github.com/OS4CSAPI/connected-systems-go/issues/4) — *Research / spike: should observation numeric fields accept `"NaN"`, `null`, or a `nilValue` sentinel?*
- **Type**: Research spike — but live testing surfaced a material defect: `nilValues` is plumbed through the data model and round-trips on GET, but is **never consulted** by the result validator.
- **Test surface**: HEAD deployment `https://129-80-248-53.sslip.io/csapi-go-head/` (commit `4b994212`).
- **Evidence directory**: `docs/research/evidence/issue-004/`
- **Date**: 2026-04-30

---

## 1. Load-bearing claims (from issue body)

| ID | Claim | Verdict |
|----|-------|---------|
| C1 | Server rejects `"baro_altitude": "NaN"` (string) for a numeric field. | **Confirmed.** 400 `result.baro_altitude must be a number` |
| C2 | OSH SensorHub accepts `"NaN"` leniently. | Plausible, not re-tested. Spec idiom supports it (see §2.1). |
| C3 | Publisher workaround substitutes `0.0` and loses semantic meaning. | **Confirmed** (T7 succeeds; the data integrity loss is real). |
| C4 | Issue is "minor / research spike". | **Refuted.** Live testing shows the cs-go schema already declares a `nilValues` table on every Quantity component (per spec), but the validator silently ignores it. This is a **functional gap**, not a research question. |

---

## 2. Verification

### 2.1 Spec analysis — OGC 23-002 + SWE Common 3.0

The bundled OAS31 (`.tmp-csapi-part2.yaml`) repeatedly shows the canonical Quantity schema with `nilValues`:

```yaml
- name: temp
  type: Quantity
  uom: { code: Cel }
  nilValues:
    - reason: http://www.opengis.net/def/nil/OGC/0/missing
      value: NaN
    - reason: http://www.opengis.net/def/nil/OGC/0/BelowDetectionRange
      value: '-Infinity'
    - reason: http://www.opengis.net/def/nil/OGC/0/AboveDetectionRange
      value: +Infinity
```

The SWE Common Data Model attaches a `nilValues: [(reason, value), …]` table to scalar components. Because RFC 8259 forbids the literals `NaN` / `Infinity` / `-Infinity` as bare numeric values, the **publisher convention** in JSON encoding is to send the **string form** `"NaN"`, `"-Infinity"`, `"+Infinity"`. The server is then expected to look up the string against the field's `nilValues` table to determine the reason for the missing value.

**Therefore:** sending `"NaN"` for a numeric field is **not** a wrong-type input that should be rejected outright; it is the spec-idiomatic carrier for "no data, reason: missing", and it must be matched against the per-field `nilValues` table before any type rejection.

Evidence: [docs/research/evidence/issue-004/spec-extract-2026-04-30.md](../evidence/issue-004/spec-extract-2026-04-30.md)

### 2.2 Empirical reproduction (HEAD `4b994212`)

Full transcript: [docs/research/evidence/issue-004/nan-tests-head-2026-04-30.txt](../evidence/issue-004/nan-tests-head-2026-04-30.txt)

Setup datasets:
- **DS_PLAIN** — Quantity field, no `nilValues` declared.
- **DS_NIL** — Quantity field with `nilValues: [{reason: missing, value: "NaN"}]`.
- **DS_OPT** — Quantity field with `optional: true`.
- **DS_SENT** — Quantity field with `nilValues: [{reason: missing, value: -999}]`.

| # | Input on `result.baro_altitude` | DS | HTTP | Body | Verdict |
|---|---------------------------------|----|------|------|---------|
| T1 | `"NaN"` (string) | PLAIN | 400 | `result.baro_altitude must be a number` | reject |
| T2 | `"NaN"` (string) | **NIL** (declares NaN as nilValue) | **400** | same | **defect — declared nilValue ignored** |
| T3 | `null` | PLAIN | 400 | `must be a number` | reject |
| T4 | `null` | NIL | 400 | `must be a number` | reject (no special handling) |
| T5 | (omitted) | PLAIN | 400 | `is required by datastream schema` | OK |
| T6 | `-999.0` | PLAIN | **201** | — | sentinel passes by coincidence (numeric) |
| T7 | `0.0` (current workaround) | PLAIN | 201 | — | works but semantically wrong |
| T8 | bare `NaN` (invalid JSON) | PLAIN | 400 | `invalid character 'N'` | correct (RFC 8259) |
| T9 | bare `Infinity` (invalid JSON) | PLAIN | 400 | `invalid character 'I'` | correct |
| T10 | `true` (boolean) | PLAIN | 400 | `must be a number` | OK |
| T11 | extra field `undeclared:"hi"` | PLAIN | **201** | stored verbatim | **open schema** — extra fields silently kept |
| T12 | `-999` | **SENT** (declares -999 as nilValue) | 201 | — | passes Quantity check; nilValues not consulted |
| T13 | GET DS_NIL | — | 200 | `nilValues` round-trips intact | data model plumbed through |
| T14 | (omitted on `optional:true`) | OPT | **201** | — | `optional` IS honoured |
| T15 | `null` on `optional:true` | OPT | **400** | `must be a number` | **`optional` does not cover `null`** — asymmetry |
| T16 | GET stored observations | — | 200 | round-trip OK; T7 stored as `0`, T11 kept extra field | |

### 2.3 Source attestation — `validateDataComponentValue`

`internal/api/observation_schema_validation.go` (HEAD `4b994212`):

```go
case "quantity":
    if !isNumber(value) {
        return fmt.Errorf("%s must be a number", path)
    }
    return nil

…

func isNumber(v any) bool {
    _, ok := v.(float64)
    return ok
}
```

**Three observations:**

1. The `quantity` branch checks ONLY `float64`. It does not check the field's `component.NilValues` table.
2. The `DatastreamDataComponent` struct in `internal/model/domains/datastream.go` (line 235) has the field plumbed: `NilValues []DatastreamNilValue \`json:"nilValues,omitempty"\``. It survives JSON round-trip (T13 confirms). It just isn't read.
3. `grep -rn NilValues internal/api/` returns **zero hits**. The validator has no awareness of nilValues at all.

The same pattern applies to `count`, `text`, `boolean`, `time`, etc. — all leaf scalars enforce strict JSON-type checks with no nilValue fallback.

### 2.4 The `optional` / `null` asymmetry (T14 vs T15)

`optional: true` is implemented as: "if the field is **absent** from the result, it is OK". But sending the field with value `null` does NOT satisfy this — control reaches the `quantity` branch with `value = nil`, the `float64` type assertion fails, and the validator rejects with `must be a number`.

This is the same "asymmetric strictness" pattern documented for issue #3 (`resultTime` vs `phenomenonTime`): two ways of expressing "no value here" produce two different outcomes, and clients can't infer one from the other.

### 2.5 Open vs closed schema (T11)

T11 sends an extra `undeclared:"hi"` field that is not in the datastream's `resultSchema.fields`. The server **accepts it (201)** and **persists it verbatim** — GET observations returns the extra field intact. The validator's `datarecord` branch only iterates declared fields; unknown fields are not checked.

This may be a deliberate forward-compat extension policy, but it is undocumented. It also interacts badly with publisher refactors: a typo in a field name (`baro_altitde` instead of `baro_altitude`) will be silently accepted as an extra field while the declared field is reported missing — confusing diagnostics.

### 2.6 Cross-fork comparison

The parent fork `SomethingCreativeStudios/connected-systems-go` ships the **same file** (`internal/api/observation_schema_validation.go`, sha `2bbb6202aea6615e7b53a1d74c6344441feb6e96` per GitHub code search). Same defect upstream — not HEAD-only.

### 2.7 Re-running the recurring anti-pattern audit (lessons learned from #1–#3)

The codebase-wide pattern from prior evals — **type-assert-and-discard** — does NOT directly apply to the validator (which uses `if !ok { return error }` correctly). However, a sibling pattern is at play here: **schema fields plumbed-through-but-not-consumed**. `NilValues`, `Constraint`, and `Updatable` are all declared on `DatastreamDataComponent` and round-trip on GET, but a quick grep shows none of them are read in the validator. Likely follow-up audit area, not just for this issue.

---

## 3. Reproduction summary

```bash
# Declare a datastream with nilValues per spec
curl -X POST $BASE/systems/$SYS/datastreams -d '{
  "...": "...",
  "schema": {"obsFormat":"application/json","resultSchema":{"type":"DataRecord",
    "fields":[{"name":"baro_altitude","type":"Quantity","uom":{"code":"m"},
      "nilValues":[{"reason":"http://www.opengis.net/def/nil/OGC/0/missing","value":"NaN"}]}]}}}'

# Send the spec-idiomatic missing-value carrier
curl -X POST $BASE/datastreams/$DS/observations -d '{
  "resultTime":"2026-04-30T12:00:00Z",
  "result":{"baro_altitude":"NaN"}}'
# → HTTP 400 {"error":"... result.baro_altitude must be a number"}
# Expected: HTTP 201, observation stored with reason="missing"
```

---

## 4. Verdicts

| Surface | Behaviour | Spec-conformant? | Defect class |
|---------|-----------|------------------|--------------|
| `"NaN"` string with no `nilValues` declared | Reject | Defensible (no declared mapping) | OK |
| `"NaN"` string with matching `nilValues` declared | **Reject** | **No** | **P2 — feature broken; declared field ignored** |
| `null` for required numeric | Reject | OK | none |
| `null` for `optional:true` numeric | Reject | Ambiguous | P3 — asymmetric vs. omission |
| Bare `NaN`/`Infinity` (literal) | Reject | Yes (RFC 8259) | none |
| Numeric sentinel matching declared `nilValues.value` | Accept (by type coincidence) | Partly | P3 — accepted but reason metadata not surfaced |
| Extra unknown field in result | Accept, persist verbatim | Permissive | P3 — undocumented open-schema policy |
| Round-trip of `nilValues` on GET datastream | OK | Yes | none |

---

## 5. Reasoning

The issue body asked: *"should the server accept NaN, or null, or use 0.0, or implement nilValue?"*. The spec already chose: **nilValues**. cs-go already has the schema plumbing for it. The defect is that the validator was written without consulting the table.

The minimum correct behaviour for a leaf scalar component validator is:
1. If the value is the expected JSON type (e.g. `float64` for Quantity), accept.
2. **Else**, look up the value (in its given JSON form) against `component.NilValues[*].Value`. If a match is found, accept and (optionally) record the matching `reason` URI for downstream use.
3. Else, reject.

This makes the publisher's `"NaN"` string work end-to-end whenever the datastream declares it, **without** changing the rejection of arbitrary wrong-typed inputs.

The `null`-vs-omitted asymmetry on `optional:true` should be resolved by treating `null` as equivalent to omission. Repeats the diagnosis from issue #3 (#19): two ways of expressing absence should converge.

---

## 6. Recommendation

Adopt **strict-with-nilValue-fallback**:

1. **Implement nilValues consultation in `validateDataComponentValue`.** For every leaf scalar branch (`quantity`, `count`, `time`, `category`, `text`, `boolean`), wrap the type check in a helper:
   ```go
   func acceptsValue(component *domains.DatastreamDataComponent, value any, typeOK bool) bool {
       if typeOK { return true }
       if component == nil { return false }
       for _, nv := range component.NilValues {
           if matchesNilValue(nv.Value, value) { return true }
       }
       return false
   }
   ```
   Where `matchesNilValue` decodes `nv.Value` (a `json.RawMessage`) and compares to the input value (string-to-string, number-to-number).

2. **Treat `null` on `optional: true` as absence.** In the `datarecord` branch, when `fieldVal == nil` and `field.Optional != nil && *field.Optional`, treat it as if the field were missing.

3. **Document the result-schema validation contract** in `docs/api/observations.md`:
   - Required vs optional fields, semantics of `null`.
   - `nilValues` mechanism, with an example showing `"NaN"` round-trip.
   - The current open-schema behaviour for unknown fields (or close it).

4. **Defer numeric leniency.** Same conclusion as #3 — do not bypass the spec's mechanism.

5. **Update OSHConnect-Python** (downstream) to send `"NaN"` (string) directly once the server fix lands; remove the `→ 0.0` substitution and add a `nilValues` declaration to the OpenSky datastream schema.

---

## 7. Open questions

- Should `Constraint` (interval / allowed-tokens) be enforced too, or is the "plumbed-but-unused" pattern intentional? Cross-issue audit candidate.
- Should the open-schema behaviour (T11) be tightened (reject extra fields) or formalized as an extension hook? **Recommend formalize**: log a warning, surface in response Link, but accept — same as how the spec treats x-extensions in the OAS document.
- Is there appetite to attach the matched `reason` URI to the stored observation (e.g. `result._nilReasons: {"baro_altitude": "...missing"}`) so downstream consumers can distinguish missing from real-zero? **Recommend yes** as a v1.1 feature; not blocking for this issue.

---

## 8. Follow-up surfaces

| # | Type | Title | Priority | Status |
|---|------|-------|----------|--------|
| F1 | Bug | `nilValues` declared on Quantity / scalar components is ignored by the result validator — spec-idiomatic `"NaN"` / `"-Infinity"` strings are rejected as wrong-type | P2 (functional gap) | filed below |
| F2 | Bug | `null` on a numeric field with `optional: true` is rejected, while the same field omitted is accepted — asymmetric absence | P3 (UX / consistency) | filed below |
| F3 | Enhancement | Document result-schema validation contract: required/optional, null semantics, nilValues, open-schema policy for extra fields | P3 (docs) | filed below |

---

## Changelog

- 2026-04-30 — Initial evaluation. Static analysis of `validateDataComponentValue` + 16 live test cases on HEAD + cross-fork verification + OGC 23-002 OAS31 schema review (the spec example explicitly shows `nilValues: [{reason: missing, value: NaN}, …]` for Quantity). Findings F1/F2/F3 to be filed as separate issues.
