# Issue #10 — SensorML `documents` array silently dropped

**Issue URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/10
**Type:** Reported as `bug`, no explicit severity in body
**Verdict:** **VALIDATED. Real defect. Spec field name verified. Live-isolated to a single class of struct-tag mismatch. Recommend P2 (defensible as P1 — silent data loss with 2xx status).**

This is the **second** issue in the run (after #8) where the spec-and-live evidence sides with the report on the main thrust. Unlike #8, the report's framing here is correct in every respect; minor refinements concern severity and scope, not validity.

---

## 1. The literal claim

> Server accepts `documents` array on POST/PUT (HTTP 200/201) but silently drops it. Subsequent GET responses contain `identifiers`, `classifiers`, `contacts`, `position` — but no `documents` array at all.

Confirmed live on HEAD (`https://129-80-248-53.sslip.io/csapi-go-head/`). See [`evidence/issue-010/live-test-2026-04-30.txt`](../evidence/issue-010/live-test-2026-04-30.txt).

| | Body property | HTTP | GET roundtrip preserves it? |
|---|---|---|---|
| **T1** | `"documents": [...]` (spec-correct) | 201 Created | **No** — entirely absent |
| **Control** | `"documentation": [...]` (cs-go's tag) | 201 Created | **Yes** — full payload returned |

## 2. Spec authority — the property name is `documents`, not `documentation`

The canonical OGC SensorML JSON encoding `DescribedObject.json` schema (in `opengeospatial/ogcapi-connected-systems/sensorml/schemas/json/`) literally defines:

```json
"documents": {
  "description": "Additional documentation about the asset",
  "type": "array",
  "items": { "$ref": "Document.json" }
}
```

`PhysicalSystem.json` inherits from `AbstractPhysicalProcess.json` → `AbstractProcess.json` → `DescribedObject.json`. None of the intermediate schemas redefine, alias, or rename the property. The canonical examples (`weather_station_system.json` and `examples/spec/documents.json`) both use `"documents"`. Plural-noun pattern matches sibling fields `contacts`, `identifiers`, `classifiers`. The schema description even says *"Additional documentation about the asset"* in prose — but the property identifier itself is `documents`. See [`evidence/issue-010/spec-authority-2026-04-30.txt`](../evidence/issue-010/spec-authority-2026-04-30.txt).

The issue body says:
> Relevant SensorML spec reference: OGC SensorML 2.1, AbstractProcess → documentation / DocumentList

That sentence is the **only** place in the issue where `documentation` appears as a possible spec name — and it refers to the *XML* element name. The XML element is `<sml:documentation>` containing a `<DocumentList>`. When SensorML was given a JSON encoding, the SWG renamed the property to the array-friendly plural `documents` (matching `contacts`, `identifiers`, etc.). The XML element name is not the JSON property name.

This XML-to-JSON naming difference is almost certainly how the bug entered cs-go: someone read the XML schema or older SensorML documentation and used `documentation` as the JSON tag.

## 3. Static evidence — 9 affected struct fields, 5 files

```
internal/model/common_shared/history.go:97   `json:"documentation,omitempty"`
internal/model/domains/deployment.go:34      `json:"documents,omitempty"`     ← only correct one
internal/model/domains/deployment.go:112     `json:"documentation,omitempty"`
internal/model/domains/procedure.go:32       `json:"documentation,omitempty"`
internal/model/domains/procedure.go:122      `json:"documentation,omitempty"`
internal/model/domains/system_event.go:22    `json:"documentation,omitempty"`
internal/model/domains/system.go:41          `json:"documentation,omitempty"`
internal/model/domains/system.go:121         `json:"documentation,omitempty"`
internal/model/domains/system.go:151         `json:"documentation,omitempty"`
```

8 of 9 use the wrong literal. The single correct tag (`deployment.go:34`) is the GORM DB struct, but the *output* SML struct for /deployments (line 112) still uses `documentation`, so /deployments is broken on the wire even though the DB column is correctly named. This internal inconsistency is itself worth flagging — it suggests partial awareness of the issue at some point.

## 4. Why the data is dropped, not preserved

Go's `encoding/json` defaults:
- **Unmarshal**: unknown JSON keys are silently ignored unless `DisallowUnknownFields()` is set (cs-go does not set it).
- **Marshal**: nil/empty + `omitempty` → field is omitted from output entirely.

So:
1. Client POSTs `"documents": [...]`. The Go SystemSML struct's tag is `documentation`, not `documents`. The unmarshaller skips the unknown key. `Documentation` stays `nil`.
2. The system persists with a null/empty `documents` JSONB column.
3. On GET, `Documentation` is nil + `omitempty`, so no field is emitted.
4. HTTP 201 is returned because everything else in the payload deserialized fine — there is no signal to the client that part of the payload was discarded.

The "silent data loss with a 2xx status" pattern is the most insidious flavor of this class of bug because clients cannot detect it locally and cannot retry their way out of it.

## 5. Scope — all four SensorML-accepting resource types

| Resource | Affected? | Evidence |
|---|---|---|
| `/systems` | **Yes (live confirmed)** | T1 vs control above |
| `/deployments` | **Yes (static)** | deployment.go:112 SML struct uses `documentation`; line 34 DB struct uses `documents`, but the wire format is determined by the SML struct |
| `/procedures` | **Yes (static)** | procedure.go:32 + :122 both use `documentation` |
| `/system-events` | **Yes (static)** | system_event.go:22 uses `documentation`; severity depends on whether clients send SML for events (likely lower-traffic path) |
| `Event.documentation` (history items) | **Yes (static)** | history.go:97 — needs separate spec verification of Event.json's property name; out of scope for #10 but same fix pattern |

The issue's claim that "All 37 systems on the Go server are missing their photo thumbnails despite the publisher sending correct SensorML with `documents` on bootstrap" is consistent with the live evidence and the static scope: the publisher is doing the right thing; cs-go is silently dropping the data.

## 6. Severity assessment

The issue is filed with `bug` label and no explicit severity. Recommend **P2 (Major)**, with a defensible **P1 (Critical)** case:

- **For P2**: localized to one field; underlying types and pipeline work correctly; affects optional documentation rather than core feature-of-interest data.
- **For P1**: silent partial-data loss (no error to client); affects every spec-correct client; affects every resource type that accepts SensorML; documented operational impact ("All 37 systems missing thumbnails"); there is no client-side workaround that doesn't require the publisher to deliberately violate the spec.

I'm landing on **P2 firm** with a recommendation that the maintainer consider P1 given the silent-loss-with-2xx pattern. Either is defensible and the fix path is identical.

## 7. Recommended fix

A 9-line search-and-replace across 5 files:

```diff
- json:"documentation,omitempty"
+ json:"documents,omitempty"
```

Plus regression coverage: a unit test that round-trips a `documents` array through the SystemSML / DeploymentSML / ProcedureSML / EventSML structs.

**Important migration note**: any data already persisted under the wrong tag (e.g., the control test row this evaluation just created, plus any client that already adapted to cs-go's wrong tag spelling) will be inaccessible after the fix unless a column migration is performed. The DB column for /deployments is named `documents` (per gorm tag); for /systems and /procedures the column name needs verification because the gorm tag also says `documentation`. A small migration to rename JSONB columns and re-key persisted JSON values is part of the complete fix.

## 8. No follow-up issues filed

All four affected resource types share a single class of fix. The same evaluation applies to /deployments, /procedures, /system-events, and Event.documentation. Filing four separate issues would be noise. I will note the broader scope in the comment on #10 so the maintainer can address them together. If the maintainer prefers separate tickets, they can split.

## 9. Methodology notes

Patterns and rules confirmed by this evaluation:

1. **Spec-name vs. encoding-name vs. struct-name vs. column-name are four separate concepts.** XML element names ≠ JSON property names, even when the underlying schema is conceptually identical. Always verify the *encoding-specific* name from the canonical encoding schema, not the prose description, not the XML element name, and not a sibling field.
2. **Issue's own spec citation can be a misleading translation.** The issue body cites "OGC SensorML 2.1, AbstractProcess → documentation / DocumentList" — that's the XML-encoding name. The cite is technically accurate but does not establish what the *JSON encoding* property name is. The issue author's own framing inadvertently looks like it might support the cs-go tag spelling, but the actual JSON schema disagrees.
3. **Internal inconsistency is a smell of half-awareness.** cs-go has the correct tag in *one* place (deployment.go:34, the DB struct) and the wrong tag everywhere else. This signals that someone, at some point, knew the right name but did not propagate the fix to all touch points.
4. **Control tests with the wrong-but-matching name isolate root cause cheaply.** Posting once with `documents` and once with `documentation` and comparing roundtrip behavior took 30 seconds and conclusively isolated the bug to the json tag literal. Use this pattern for any "field silently dropped" claim.
5. **Silent partial loss with 2xx status is a severity multiplier.** Keep this in mind for severity recommendations on similar future findings.
