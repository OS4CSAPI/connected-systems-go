# Issue #12 — Static analysis (2026-04-30)

Repo @ `c9e4fcf` (`origin/main`).

## 1. Datastream model has no `unique_identifier` field

`internal/model/domains/datastream.go`:

```go
type Datastream struct {
    Base
    Name        string `gorm:"type:varchar(255);not null" json:"name"`
    Description string `gorm:"type:text" json:"description,omitempty"`
    ValidTime   *TimePeriod ...
    Formats     pq.StringArray ...
    SystemLink         *Link ...
    OutputName         string ...
    ProcedureLink      *Link ...
    DeploymentLink     *Link ...
    FeatureOfInterest  *Link ...
    SamplingFeatureLink *Link ...
    ObservedProperties []ObservedProperty ...
    PhenomenonTime     *TimePeriod ...
    PhenomenonTimeInterval string ...
    ResultTime         *TimePeriod ...
    ResultTimeInterval string ...
    Type       string ...
    ResultType string ...
    Live       *bool ...
    Schema     ObservationSchema ...
    Links      Links ...
    SystemID            *string `gorm:"type:varchar(255);index" json:"-"`
    ProcedureID         *string `gorm:"type:varchar(255);index" json:"-"`
    DeploymentID        *string `gorm:"type:varchar(255);index" json:"-"`
    FeatureOfInterestID *string `gorm:"type:varchar(255);index" json:"-"`
    SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`
    Systems []System `gorm:"many2many:..." json:"-"`
}

func (Datastream) TableName() string { return "datastreams" }
```

Datastream embeds **only `Base`** (`{ID string; CreatedAt; UpdatedAt}` — no uniqueIndex). It does **not** embed `CommonSSN`, which is the type that carries `UniqueIdentifier UniqueID gorm:"type:varchar(255);uniqueIndex" json:"uid"`.

`grep` for embedders of `CommonSSN` (`grep -RnE '^\s*CommonSSN\s*$' internal/model/domains/`):
- `control_stream.go`
- `deployment.go`
- `feature.go`
- `procedure.go`
- `property.go`
- `sampling_feature.go`
- `system.go`

`Datastream` and `Observation` are **not** in this list. There is no `UniqueIdentifier` or `gorm:"...uniqueIndex"` field anywhere on `Datastream`.

`AutoMigrate` (`internal/repository/repository.go::AutoMigrate`) migrates `&domains.Datastream{}` directly, so the live DB schema reflects the struct above: no `unique_identifier` column on `datastreams`.

The issue's stated premise — "DB-level `UNIQUE(unique_identifier)` on `datastreams` enforces global uniqueness" — is **not present in the current code**.

## 2. Git history — the constraint was real but already removed

`git log --oneline --all -- internal/model/domains/datastream.go`:

```
1562201 update datastreams
dacae7b Bug fixes and clean up
f2cf1c3 Adding the other resources along with e2e tests
```

`git log -G "CommonSSN" --oneline --all -- internal/model/domains/datastream.go`:

```
1562201 update datastreams
f2cf1c3 Adding the other resources along with e2e tests
```

Diff at `1562201`:

```diff
--- a/internal/model/domains/datastream.go
+++ b/internal/model/domains/datastream.go
-       CommonSSN
+       Name        string `gorm:"type:varchar(255);not null" json:"name"`
+       Description string `gorm:"type:text" json:"description,omitempty"`
```

So the issue's reported behaviour was historically real: prior to commit `1562201`, `Datastream` embedded `CommonSSN`, which contributed a `unique_identifier varchar(255)` column with a `uniqueIndex` (single-column, table-scoped UNIQUE). Commit `1562201` "update datastreams" replaced the embedding with explicit `Name`/`Description` fields. With GORM `AutoMigrate`, this **does not drop the column from existing databases**, but new deployments created from `c9e4fcf` will not have the column at all.

Two implications:

1. The defect the issue describes was real on whatever revision the OSHConnect-Python publisher tested against, but cs-go HEAD has already moved past it (and beyond — see point 3).
2. Long-running deployments that started on `f2cf1c3..1562201^` and were never wiped may still carry the column, because `AutoMigrate` is additive. Whether the live HEAD endpoint inherits a stale column matters — see live testing for the empirical answer.

## 3. The 1562201 cleanup was incomplete: dangling SQL reference at `datastream_repository.go:164`

`internal/repository/datastream_repository.go`, in the params-driven filter builder:

```go
if len(params.IDs) > 0 {
    query = query.Where("id IN ? OR unique_identifier IN ?", params.IDs, params.IDs)
}
```

This SQL refers to a column that the `Datastream` struct no longer declares. On any deployment built from `c9e4fcf` where `AutoMigrate` did not back-fill the column (i.e. fresh installs), any `?id=` query against `/datastreams` will produce a SQL-level "column does not exist" error, surfacing as HTTP 500.

This is **collateral damage from the partial cleanup in 1562201**. It is a real defect, but it is **not** the defect issue #12 describes — in fact it is roughly the opposite. Issue #12 says "uniqueness is too strict"; this dangling reference says "the column it references is gone". I record it here as an adjacent finding and recommend filing it as a separate issue.

## 4. Spec — Datastream JSON encoding has no `uid` property

CSAPI Part 2 canonical schemas (`opengeospatial/ogcapi-connected-systems@master`, `api/part2/openapi/schemas/json/`):

`baseStream.json` properties: `id`, `name`, `description`, `validTime`, `formats`. Required: `id`, `name`, `formats`. **No `uid`.**

`dataStream.json` (`allOf [baseStream.json, { ... }]`) adds: `system@link`, `outputName`, `procedure@link`, `deployment@link`, `featureOfInterest@link`, `samplingFeature@link`, `observedProperties`, `phenomenonTime`, `phenomenonTimeInterval`, `resultTime`, `resultTimeInterval`, `type`, `resultType`, `live`, `schema`, `links`. Required: `name`, `system@link`, `observedProperties`, `phenomenonTime`, `resultTime`, `resultType`, `live`. **No `uid`.**

By contrast, `system.json` (CSAPI Part 1) does carry `uid` via the corresponding base. So `uid` is a System-level concept; the canonical spec does not assign datastreams a UID URN at all. They are identified by their server-side `id` only and are intrinsically children of a system.

This is a corroborating point: even the issue's *proposed* fix (Option A — composite uniqueIndex on `(system_id, unique_identifier)`) imports a System-level concept onto Datastream. A spec-faithful Datastream model has no `uid` field at all. The current cs-go HEAD model — Datastream identified by `id`, no UID — matches the spec.

## 5. Handler does not validate UID

`internal/api/datastream_handler.go::CreateDatastream` (lines 118-150):

```go
systemID := chi.URLParam(r, "systemId")
...
contentType := r.Header.Get("Content-Type")
datastream := &domains.Datastream{}
if err := fc.Deserialize(contentType, body, datastream); err != nil { ... }
datastream.SystemID = &systemID
if err := h.repo.Create(datastream); err != nil { 500 }
w.Header().Set("Location", ...)
w.WriteHeader(http.StatusCreated)
```

No UID inspection, no uniqueness check at the handler level. Any `uid` field in the request body has no matching struct field on `Datastream`, so `json.Unmarshal` silently drops it. A `properties.uid` nested under `properties` is also silently dropped — Datastream has no `Properties` field.

Net: cs-go HEAD has no enforcement of UID uniqueness on `datastreams` whatsoever (within or across systems), and no round-trip of a UID at all. This is structurally different from what issue #12 reports.
