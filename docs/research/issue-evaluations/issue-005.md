# Issue #5 — Strict schema validation: requiring all declared fields in observation results

- **Issue**: [OS4CSAPI/connected-systems-go#5](https://github.com/OS4CSAPI/connected-systems-go/issues/5) — *Research / spike: should observation results be strictly validated against the datastream schema?*
- **Type**: Research spike. Live testing **refutes the framing**: cs-go does NOT strictly require all declared fields — it requires fields **not marked `optional: true`**. The publisher's stated workaround was unnecessary.
- **Test surface**: HEAD deployment `https://129-80-248-53.sslip.io/csapi-go-head/` (commit `4b994212`).
- **Evidence**: `docs/research/evidence/issue-005/`
- **Date**: 2026-04-30

---

## 1. Load-bearing claims (from issue body)

| ID | Claim | Verdict |
|----|-------|---------|
| C1 | Server requires EVERY field declared in the schema to be present in EVERY observation. | **Partially refuted.** The server requires every **non-optional** field. Per-field `optional: true` flag exists, is consumed by the validator (lines 92, 115 of `observation_schema_validation.go`), and works end-to-end (T2 below). |
| C2 | Publisher had to duplicate `resultTime` into `result.timestamp` to satisfy the validator. | **Confirmed as a workaround**, but the workaround was avoidable: declaring `timestamp` with `"optional": true` in the datastream schema yields the desired behaviour. |
| C3 | OSH SensorHub silently accepts subset payloads. | Plausible but irrelevant once the optional mechanism is used correctly — see §5. |
| C4 | Issue is "P3 minor research spike". | Validated. The headline question is documentation/discoverability, not a server defect. The interesting sibling defects (null-vs-omit asymmetry, open-schema for extras) are already filed under #4's follow-ups (#22, #23). |

---

## 2. Verification

### 2.1 Spec analysis

OGC 23-002 Part 2 §9.7 defines observation result validation in terms of the SWE Common Data Model. SWE Common 3.0 §7.4 (DataRecord component) defines field membership via `field` elements that each carry an `optional` boolean (default `false`). The server's behaviour — required-by-default, opt-out via `optional: true` — matches this model precisely.

The XML Schema concept `minOccurs="0"` is **not** the JSON encoding mechanism. SWE Common JSON encoding uses the `optional` boolean. cs-go's `DatastreamDataComponent.Optional *bool` field implements the spec mechanism directly.

### 2.2 Source attestation

`internal/api/observation_schema_validation.go` (HEAD `4b994212`):

```go
case "datarecord":
    obj, ok := value.(map[string]any)
    if !ok { return fmt.Errorf("%s must be an object", path) }
    for _, field := range component.Fields {
        if field.Name == "" { continue }
        fieldVal, exists := obj[field.Name]
        if !exists {
            if field.Optional != nil && *field.Optional { continue }
            return fmt.Errorf("%s.%s is required by datastream schema", path, field.Name)
        }
        if err := validateDataComponentValue(&field.DatastreamDataComponent, fieldVal, path+"."+field.Name); err != nil {
            return err
        }
    }
    return nil
```

`internal/model/domains/datastream.go:225`: `Optional *bool \`json:"optional,omitempty"\``. The same pattern repeats in the `vector` branch at line 115 for `coord.Optional`.

`grep -rn ".Optional" internal/api/` → 2 hits (the two consumption sites above). The mechanism is fully wired for `datarecord` and `vector` field omission.

**What the validator does NOT do** (by design or oversight):
- It does not iterate over the input object's keys to detect undeclared fields → extra fields are silently accepted.
- It does not treat `null` as equivalent to omission for `optional: true` fields → asymmetric absence (filed as #22 from issue #4).
- `datachoice`, `dataarray`, `matrix` branches do not have an `optional` consumer (no equivalent loop with per-field opt-out).

### 2.3 Empirical reproduction (HEAD `4b994212`)

Full transcript: [docs/research/evidence/issue-005/strict-tests-head-2026-04-30.txt](../evidence/issue-005/strict-tests-head-2026-04-30.txt).

Setup: two datastreams with identical 6-field schemas (`timestamp`, `latitude`, `longitude`, `altitude`, `heading`, `velocity`). DS_ALLREQ has no `optional` flags; DS_TS_OPT marks `timestamp` as `optional: true`.

| # | DS | Payload | HTTP | Verdict |
|---|----|---------|------|---------|
| T1 | DS_ALLREQ | publisher's omitted-`timestamp` payload | 400 `result.timestamp is required` | spec-conformant given declared schema |
| T2 | DS_TS_OPT | **same payload** | **201** | **proves opt-out mechanism works** |
| T3 | GET DS_TS_OPT | — | 200 with `timestamp -> True`, others `(absent)` | `optional` round-trips through GET |
| T4 | DS_ALLREQ | full 6-field workaround | 201 | sanity baseline |
| T5 | DS_ALLREQ | full payload + extra `undeclared:"x"` | 201 | extras silently accepted (open schema) |
| T6 | DS_ALLREQ | typo `latitudee` (declared field is `latitude`) | 400 `result.latitude is required` | typo'd field silently kept as extra; declared field reports missing — confusing diagnostic |

T6 is the key sibling finding: under the open-schema policy, a typo in a publisher's field name produces a "field is required" error for the declared name, while the typo'd field is silently kept. The publisher cannot tell from the error message whether they sent the wrong name or whether the field truly went missing in their pipeline. This is already covered by #23 (document the contract) but worth elevating in the comment back to #5.

### 2.4 Cross-fork

Same file (`internal/api/observation_schema_validation.go`) ships in `SomethingCreativeStudios/connected-systems-go` (sha `2bbb6202`). Same `Optional` consumption sites. No divergence.

### 2.5 Sibling silent-failure audit

Per the issue-003 / issue-004 methodology pattern: when a research spike asks "is the server too strict?", check whether *adjacent* surfaces are too **lenient** in inconsistent ways. The audit here:

| Surface | Behaviour | Notes |
|---------|-----------|-------|
| Required field omitted (`datarecord`) | Reject 400 ✓ | spec-conformant |
| Required field omitted (`vector` coord) | Reject 400 ✓ | spec-conformant |
| Optional field omitted | Accept 201 ✓ | spec-conformant |
| Optional field = `null` | **Reject 400** | **#22 — asymmetric absence** |
| Extra undeclared field | Accept 201, persist verbatim | **#23 — undocumented open-schema** |
| Field with declared but unconsumed `nilValues` | Reject 400 if non-numeric | **#21 — nilValues plumbed-but-unused** |
| `dataarray`/`matrix` with `optional` element | unknown — branch has no opt-out logic | possible gap; out of scope here |
| `datachoice` with `optional` selector | unknown — branch not yet inspected | possible gap; out of scope here |

The `datachoice`/`dataarray` gaps are **plausible but speculative** — they would only matter if a publisher declared `optional: true` on a non-leaf component, which the spec does not clearly prescribe. Flagging as an audit candidate, not filing a separate issue.

---

## 3. Reproduction summary

```bash
# Same payload, two schemas:
PAYLOAD='{"resultTime":"2026-04-30T12:00:00Z","result":{"latitude":40.0,"longitude":-74.0,"altitude":10500.0,"heading":180.0,"velocity":250.0}}'

# Schema with timestamp required (default) → 400
curl -X POST $BASE/datastreams/$DS_ALLREQ/observations -d "$PAYLOAD"
# → {"error":"... result.timestamp is required by datastream schema"} (400)

# Same schema with "optional": true on timestamp → 201
curl -X POST $BASE/datastreams/$DS_TS_OPT/observations -d "$PAYLOAD"
# → 201 Created
```

The publisher's original "duplicate `resultTime` into `result.timestamp`" workaround is **unnecessary**. The correct fix is in the **datastream schema declaration**, not in every observation payload.

---

## 4. Verdicts

| Question (from issue body) | Answer |
|----------------------------|--------|
| Q1 — What does OGC 23-002 say about result validation? | Validate against the SWE Common DataRecord schema. Required-by-default with per-field `optional` opt-out is the canonical mechanism. cs-go matches. |
| Q2 — Does SWE Common distinguish required vs optional? | Yes — `optional` boolean on each field/coord, default `false`. JSON encoding uses the same flag. |
| Q3 — Should the schema support marking fields optional? | **Already supported.** `DatastreamDataComponent.Optional *bool`, consumed by `validateDataComponentValue`. T2 confirms end-to-end. |
| Q4 — Performance implication? | Per-record map lookup per declared field — O(declared_fields) per observation, no measurable impact at any realistic schema size. |
| Q5 — How do other CSAPI servers handle partial observations? | SensorHub appears lenient (per issue body); cs-go is correct-by-spec when `optional` is declared appropriately. The interop gap is publisher-side (declare optionality), not server-side. |
| Q6 — Should extra fields be accepted, ignored, or rejected? | Currently accepted-and-persisted (open schema). Already filed as #23 (P3 docs) for either tightening or formalizing as a documented extension hook. |
| Q7 — Schema "what CAN appear" vs "what MUST appear"? | The canonical SWE Common reading is "what the datastream emits"; required-by-default with opt-outs. cs-go follows this. |

**Bottom line:** The issue's premise is mistaken; the server is not "strictly requiring all fields" — it requires non-optional fields, exactly per spec. The optional mechanism is implemented and works.

---

## 5. Reasoning

The OSHConnect-Python publisher hit this because they declared all 6 fields without `optional` and then sent some without all 6. From cs-go's perspective: spec-correct rejection. From the publisher's perspective: surprising — they expected SensorHub-style leniency. The right fix is **not** to make cs-go lenient (that would silently lose data integrity guarantees for publishers who DO want strict checks), but to:

1. **Update OSHConnect-Python's datastream definitions** to mark fields like `timestamp` (redundant with `resultTime`) as `"optional": true`.
2. **Document the optionality mechanism** in the cs-go README / observation-result-schema doc (folded into #23).

No new follow-up issues from this spike. The headline finding (the optional mechanism works — issue mis-frames it as missing) is informational, not actionable. The two genuine defects in the area (null-vs-omit asymmetry, open-schema policy) are already filed under #4's umbrella as #22 and #23.

---

## 6. Recommendation

**Close #5 as research-resolved with no code changes required.** The optional-field mechanism is implemented, spec-conformant, and works end-to-end. Action items are downstream:

- (Publisher) OSHConnect-Python should mark `timestamp` and any other resultTime-redundant fields as `optional: true` in its datastream schema, then drop the duplication workaround.
- (Server docs) The optional-field mechanism should appear in the result-schema documentation deliverable already tracked under #23.

No new follow-on issues. The sibling defects in this area are tracked under:
- **#21** — nilValues table is plumbed but unused (P2).
- **#22** — `null` on `optional: true` is rejected while omission is accepted (P3, asymmetric absence).
- **#23** — document the result-schema validation contract (P3, docs).

---

## 7. Open questions (deferred)

- Do `datachoice` / `dataarray` / `matrix` branches honour `optional` on their element/selector subcomponents? Static read of the validator suggests **no** (the `optional` consumption is only in the `datarecord` and `vector` branches). This may matter for advanced schemas; not pursued here because no observed publisher uses those constructs in the failure mode the issue describes. **Audit candidate** if/when such schemas appear.
- Should typo'd fields (T6) be detectable? The current behaviour — silent acceptance of unknown keys + "missing field" error for the declared name — is consistent with the open-schema policy but is hostile to publishers. A `?strict=true` query parameter or a server-side warning header listing unknown keys would solve it without breaking forward compatibility. **Folded into #23**'s scope.

---

## Changelog

- 2026-04-30 — Initial evaluation. Static analysis + 6 live test cases on HEAD prove the optional-field mechanism is implemented and works end-to-end. Reframes the issue from "server defect" to "publisher discoverability". No new follow-ups; relevant defects already tracked under #21/#22/#23.
