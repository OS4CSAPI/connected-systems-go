# Issue #15 — Static analysis evidence

**Date:** 2026-04-30
**Repo HEAD evaluated:** `e0cb0dcbd033638b3fd079bd6204befd0d892f07`
**Scope:** Verify (a) `Datastream.Systems` field is missing `json:"-"`,
(b) every other relationship/FK field in the codebase has the tag,
(c) sweep for any sibling/adjacent leaks the issue does not mention.

---

## 1. The cited line — `internal/model/domains/datastream.go:57`

```go
type Datastream struct {
    Base
    Name        string `gorm:"type:varchar(255);not null" json:"name"`
    ... (lines 16–49 — all properly json-tagged) ...

    // Optional normalized IDs retained server-side for easier joins/filtering.
    SystemID            *string `gorm:"type:varchar(255);index" json:"-"`  // line 51
    ProcedureID         *string `gorm:"type:varchar(255);index" json:"-"`  // line 52
    DeploymentID        *string `gorm:"type:varchar(255);index" json:"-"`  // line 53
    FeatureOfInterestID *string `gorm:"type:varchar(255);index" json:"-"`  // line 54
    SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`  // line 55

    Systems []System `gorm:"many2many:system_datastreams;"`               // line 57 — MISSING json:"-"
}
```

Confirmed by:

```text
$ git grep -n 'Systems \[\]System' -- internal/model/domains/datastream.go
internal/model/domains/datastream.go:57:        Systems []System `gorm:"many2many:system_datastreams;"`
```

The field carries only a `gorm` tag; no `json` tag is present. Go's
`encoding/json` rules (`encoding/json` package documentation, "Marshal"):
when no `json:` struct tag is present, the field is encoded under its
exported Go name (`Systems`).

The field is the back-reference for the `system_datastreams` join
table; it is not loaded by any of the read handlers (verified by
`git grep -n 'Preload.*Systems' -- internal/`), so it always
serialises as the zero value of a Go slice — which `encoding/json`
emits as `null`, not `[]`.

(Note: the issue body says "line ~70". Actual line on HEAD is **57**.
Trivial drift; substance unchanged.)

## 2. Codebase convention — every other relationship/FK field has `json:"-"`

```text
$ git grep -n 'json:"-"' -- internal/model/domains/
internal/model/domains/common.go:13:                CreatedAt time.Time `json:"-"`
internal/model/domains/common.go:14:                UpdatedAt time.Time `json:"-"`
internal/model/domains/control_stream.go:47-51:     SystemID/ProcedureID/DeploymentID/FeatureOfInterestID/SamplingFeatureID `... json:"-"`
internal/model/domains/datastream.go:51-55:         (same five FKs)               `... json:"-"`
internal/model/domains/datastream.go:141-142:       Inline / Link                  `json:"-"`
internal/model/domains/deployment.go:21,43:         ParentDeploymentID, PlatformID `... json:"-"`
internal/model/domains/feature.go:16:               CollectionID                   `... json:"-"`
internal/model/domains/procedure.go:54-55:          ControlledProperties, ObservedProperties `gorm:"many2many:..." json:"-"`
internal/model/domains/system.go:17,27,29,62:       SMLType, ParentSystemID, SystemKindID, SystemKind `... json:"-"`
internal/model/domains/system_event.go:14:          SystemID                       `... json:"-"`
```

The convention is clear and consistent: every internal join key,
foreign key, and relationship slice that exists for GORM joins gets
`json:"-"`. The issue's stated convention claim is fully borne out.

## 3. Sweep for ALL untagged relationship slices in `domains/`

```text
$ git grep -n 'gorm:' -- internal/model/domains/ | grep '\[\]\w\+ `' | grep -v 'json:'
internal/model/domains/control_stream.go:53: Systems          []System         `gorm:"many2many:system_controlstreams;"`
internal/model/domains/datastream.go:57:     Systems          []System         `gorm:"many2many:system_datastreams;"`     ← #15 target
internal/model/domains/system.go:67:         SamplingFeatures []SamplingFeature `gorm:"foreignKey:ParentSystemID;"`
internal/model/domains/system.go:70:         Controlstreams   []ControlStream   `gorm:"many2many:system_controlstreams;"`

# Plus two more found by direct many2many grep that the [] grep missed:
internal/model/domains/procedure.go:61:      Systems          []System          `gorm:"many2many:system_procedures;"`
internal/model/domains/system.go:65:          Procedures       []Procedure       `gorm:"many2many:system_procedures;"`
internal/model/domains/system.go:66:          Deployments      []Deployment      `gorm:"many2many:system_deployments;"`
internal/model/domains/system.go:69:          Datastreams      []Datastream      `gorm:"many2many:system_datastreams;"`
```

**Eight untagged relationship slices total**, in four files:

| File | Line | Field | Visible in JSON response today? |
|---|---|---|---|
| `datastream.go` | 57 | `Systems []System` | **YES** — issue #15 target |
| `control_stream.go` | 53 | `Systems []System` | YES on the same code path (controlstream handler marshals struct directly), currently not visible only because no controlstreams exist on HEAD to query |
| `procedure.go` | 61 | `Systems []System` | NO — procedure handler builds custom DTO |
| `system.go` | 65 | `Procedures []Procedure` | NO — system handler emits GeoJSON `Feature` shape via custom DTO |
| `system.go` | 66 | `Deployments []Deployment` | NO — same |
| `system.go` | 67 | `SamplingFeatures []SamplingFeature` | NO — same |
| `system.go` | 69 | `Datastreams []Datastream` | NO — same |
| `system.go` | 70 | `Controlstreams []ControlStream` | NO — same |

Why the System and Procedure handlers don't leak: their HTTP responses
are constructed manually as GeoJSON `Feature` / DTO envelopes, not by
direct `json.Marshal` of the GORM struct. Verified by:

- `GET /systems/{id}` returns `{"type":"Feature","id":"…","geometry":null,"properties":{"uid":"…","name":"…","featureType":"…"},"links":[…]}` — no `Procedures`/`Deployments`/`Datastreams`/`Controlstreams` keys, no struct passthrough.
- `GET /datastreams` items are direct `Datastream` struct marshals — hence the leak.

This means **two** files exhibit the actual visible-in-response leak:
`datastream.go:57` (visible) and `control_stream.go:53` (latent —
would be visible the moment a controlstream exists). The issue body
addresses only `datastream.go:57`. Per the issue's own scope rule
("If you find more, file separately, one issue per leaking type"),
`control_stream.go:53` should be a separate adjacent issue, not folded
into #15.

The remaining six untagged slices (in `system.go` and `procedure.go`)
are latent — they don't leak today only because the handlers happen to
build manual DTOs. They are still latent landmines for any future
handler/middleware that calls `json.Marshal` on the struct directly,
and best-practice still says they should carry `json:"-"`.
Recommendation in the eval: scope of #15 stays as-is; recommend a
separate broader hardening issue for the latent ones.

## 4. No custom `MarshalJSON` on `Datastream`

```text
$ git grep -n 'func .* MarshalJSON' -- internal/model/domains/
internal/model/domains/datastream.go:145:func (m DatastreamMessageSchema) MarshalJSON() ([]byte, error) {
internal/model/domains/datastream.go:274:func (ec DatastreamElementCount) MarshalJSON() ([]byte, error) {
```

The two `MarshalJSON` methods are on inner schema types
(`DatastreamMessageSchema`, `DatastreamElementCount`), not on
`Datastream` itself. Therefore `Datastream` is encoded by the default
reflection-based path of `encoding/json`, which honours struct tags as
described in §1. There is no custom marshaller masking the leak —
adding `json:"-"` is sufficient to fix it.

## 5. The proposed fix is exact and minimal

The issue's one-line change is:

```diff
- Systems []System `gorm:"many2many:system_datastreams;"`
+ Systems []System `gorm:"many2many:system_datastreams;" json:"-"`
```

This:

- preserves the GORM relationship (no schema change, no migration),
- prevents `encoding/json` from emitting the field,
- exactly mirrors the existing convention on every other relationship
  field in the same package.

No other changes are required for the fix to take effect on the
`/datastreams` and `/datastreams/{id}` endpoints.

## Summary

The static premise of issue #15 is fully validated. The leak is real,
the cited line and proposed fix are exact (modulo trivial line-number
drift: 57 vs. ~70), and the convention claim is true. One adjacent
finding (`control_stream.go:53`) that the issue body deliberately
excludes from scope per its own "file separately" rule, plus six
latent untagged slices in `system.go`/`procedure.go` that are not
currently visible in responses but should be hardened for future safety.
