# Issue #15 evaluation — Capital-`S` `Systems` GORM relationship field leaks into every datastream JSON response

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#15 |
| Title | Capital-S `Systems` GORM relationship field leaks into every datastream JSON response |
| Author | Sam-Bolling |
| Labels | `good first issue` (issue body also self-classifies as `code-quality`, `api-design`, `discovery-mode`, severity `P3-Minor`) |
| Repo HEAD evaluated | `e0cb0dcbd033638b3fd079bd6204befd0d892f07` |
| Live endpoint | `https://129-80-248-53.sslip.io/csapi-go-head` |
| Evaluation date | 2026-04-30 |
| **Verdict** | **KEEP — valid, narrowly scoped, accurately framed** |

---

## 1. Claim under evaluation

`internal/model/domains/datastream.go` declares
`Systems []System` as a GORM `many2many` back-reference for the
`system_datastreams` join table without a `json:"-"` tag, which causes
`encoding/json` to emit `"Systems": null` in every datastream HTTP
response. The issue proposes the one-line fix of appending
`json:"-"` to the existing struct tag.

## 2. Evaluation method

Standard rigour pass:

1. Read issue body verbatim. ✅
2. Static analysis (cited line, surrounding convention, sweep for
   adjacent leaks, custom marshaller check). ✅ See
   [static-analysis-2026-04-30.md](../evidence/issue-015/static-analysis-2026-04-30.md).
3. Spec authority (canonical OGC `dataStream`/`baseStream` schemas;
   JSON-Schema 2020-12 `additionalProperties` semantics; RFC 7493). ✅
   See [spec-authority-2026-04-30.md](../evidence/issue-015/spec-authority-2026-04-30.md).
4. Live empirical reproduction. ✅ See
   [live-test-2026-04-30.md](../evidence/issue-015/live-test-2026-04-30.md).
5. Write-up. ← this document.
6. Comment + plan + commit + push.

## 3. Findings

### 3.1 Empirical claim — confirmed exactly

`GET /datastreams?limit=1` on HEAD returns a datastream object whose
top-level keys are exactly the spec keys plus `"Systems": null`:

```json
{ … "links": [ … ], "Systems": null }
```

The leaked property is capital-`S`, value `null`, top level — verbatim
match to the issue body.

### 3.2 Static premise — confirmed (with one trivial drift)

- `internal/model/domains/datastream.go:57` carries
  `Systems []System `gorm:"many2many:system_datastreams;"`` with no
  `json:` tag. (Issue body says "line ~70"; actual is **57**.
  Substantively identical.)
- All five sibling FK fields in the same struct (lines 51–55) carry
  `json:"-"`.
- All FK/relationship fields in every other domain file in the package
  carry `json:"-"`. Convention claim verified.
- No `MarshalJSON` exists on `Datastream` itself; encoding goes through
  the default reflective path, so adding `json:"-"` is sufficient.

### 3.3 Spec authority — leak is **not** a strict violation

Authoritative `dataStream.json` and `baseStream.json` (OGC Connected
Systems Part 2 master branch):

- Define 21 properties total, none named `Systems` (any case).
- Do **not** declare `additionalProperties: false` or
  `unevaluatedProperties: false`, so per JSON Schema 2020-12 default
  semantics, additional properties are permitted.

The leak is therefore **not a strict schema violation**. It is a
code-quality / API-design defect plus a SHOULD-grade RFC 7493 §4.3
deviation ("senders that wish to be widely interoperable SHOULD only
emit … members that are defined in the protocol"). This matches the
issue body's own labelling (P3-Minor, `code-quality`, `api-design`)
exactly.

This is the symmetric counterpart of the spec analysis used in
issue #13 (silently ignoring unknown *input* properties is also
schema-permissive); the same JSON-Schema default-permissive principle
governs both directions.

### 3.4 Adjacent finding — `control_stream.go:53` (out of scope by the issue's own rule)

`internal/model/domains/control_stream.go:53` has the **identical**
untagged `Systems []System` field, and the controlstream handler uses
the same direct-`json.Marshal` code path as the datastream handler.
The leak will reproduce on any non-empty `/controlstreams` response.
HEAD currently has zero controlstreams, so the live test could not
exhibit it, but the static evidence is unambiguous.

The issue body's scope rule — "If you find more, file separately, one
issue per leaking type" — explicitly contemplates this. Recommendation:
**file a sibling issue** referencing #15. Recording here only; do
not fold into #15.

### 3.5 Latent untagged slices — `system.go:65–70`, `procedure.go:61`

Six additional untagged relationship slices (`Procedures`,
`Deployments`, `SamplingFeatures`, `Datastreams`, `Controlstreams` on
`System`; `Systems` on `Procedure`) are present but **do not leak
today**, because the System and Procedure handlers construct manual
GeoJSON `Feature` / DTO envelopes instead of marshalling the GORM
struct directly. Verified live on `GET /systems/{id}`: response has
`type/id/geometry/properties/links` only — no struct passthrough.

These are latent landmines: any future handler/middleware that calls
`json.Marshal(systemValue)` will leak. Recommendation: file a single
**hardening** issue covering all six, separate from #15 and from the
adjacent #15-sibling proposed in §3.4.

### 3.6 Severity assessment

P3-Minor is exactly right. P2 would imply functional or strict-spec
impact; neither exists. P4-Cosmetic would understate the
interoperability and convention-consistency value.

### 3.7 Suggested-fix evaluation

The proposed one-line change is exact and minimal:

```diff
- Systems []System `gorm:"many2many:system_datastreams;"`
+ Systems []System `gorm:"many2many:system_datastreams;" json:"-"`
```

- ✅ GORM tag preserved → no migration, no schema change.
- ✅ `json:"-"` makes `encoding/json` skip the field during marshal.
- ✅ Matches codebase convention exactly.
- ✅ Acceptance criteria (struct-tag grep + JSON-key absence +
  `go build` + `go test` + live `curl | jq has("Systems")`) are well
  formed and verifiable.
- ✅ Scope exclusions (no rename, no lower-cased `systems`, no sweep,
  no migration) are correct: lower-case `systems` is not in the
  canonical schema, and broader sweep belongs in separate issues.

## 4. Verdict

**KEEP. Issue #15 is a small, well-framed, well-scoped P3 with an
exact one-line fix that improves codebase consistency and
RFC 7493 interoperability without any risk.**

Recommended actions (suggestions to upstream maintainers, no edits to
cs-go from this evaluation):

1. Apply the proposed one-line change to
   `internal/model/domains/datastream.go:57` exactly as suggested.
2. File a sibling issue covering `control_stream.go:53` (the only
   other actively-leaking field; same fix; same convention).
3. Optionally file a broader hardening issue for the six latent
   `json:"-"`-missing slices in `system.go` and `procedure.go`.
4. Severity P3-Minor stands; no re-labelling needed.
5. The line-number annotation in the issue body ("~70") could be
   refreshed to "57" for accuracy, but this is cosmetic.

## 5. Cross-references

- Static analysis: [docs/research/evidence/issue-015/static-analysis-2026-04-30.md](../evidence/issue-015/static-analysis-2026-04-30.md)
- Spec authority: [docs/research/evidence/issue-015/spec-authority-2026-04-30.md](../evidence/issue-015/spec-authority-2026-04-30.md)
- Live test: [docs/research/evidence/issue-015/live-test-2026-04-30.md](../evidence/issue-015/live-test-2026-04-30.md)
- Canonical OGC schemas: <https://github.com/opengeospatial/ogcapi-connected-systems/tree/master/api/part2/openapi/schemas/json>
- RFC 7493 (I-JSON): <https://datatracker.ietf.org/doc/html/rfc7493#section-4.3>
- Related: issue #1 §2.8 T7 (original empirical surfacing); issue #13 (symmetric input-side analysis)
