# Issue #16 — Static analysis (HEAD `edc8459`)

Date: 2026-04-30. Issue cites HEAD `4b99421`; verified `git log 4b99421..HEAD --
internal/repository/datastream_repository.go internal/repository/control_stream_repository.go
internal/repository/system_repository.go internal/repository/deployment_repository.go
internal/api/deployment_handler.go internal/model/domains/` returns **zero** commits
— code state for every claim is byte-identical to the cited HEAD.

## Bug 1 — m2m join FKs not cleaned in `?cascade=true` paths

### `DatastreamRepository.Delete` — `internal/repository/datastream_repository.go:134-146`

```go
func (r *DatastreamRepository) Delete(id string, cascade bool) error {
    if !cascade {
        return r.db.Delete(&domains.Datastream{}, "id = ?", id).Error
    }
    return r.db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Where("datastream_id = ?", id).Delete(&domains.Observation{}).Error; err != nil {
            return err
        }
        return tx.Delete(&domains.Datastream{}, "id = ?", id).Error
    })
}
```

Cascade path deletes `Observation` rows (no FK; bare WHERE), then deletes the
datastream. **Never touches `system_datastreams` join.** The single FK that
exists on the relation — `fk_system_datastreams_datastream` — fires on the final
`tx.Delete(&domains.Datastream{}, ...)` whenever any `system_datastreams` row
references this datastream, which is true for every datastream created via
`POST /systems/{id}/datastreams` (the only POST route that exists for
datastreams; `internal/api/router.go:118-126` confirms there is no top-level
`POST /datastreams`).

### `ControlStreamRepository.Delete` — `internal/repository/control_stream_repository.go:134-145`

```go
func (r *ControlStreamRepository) Delete(id string, cascade bool) error {
    if !cascade {
        return r.db.Delete(&domains.ControlStream{}, "id = ?", id).Error
    }
    return r.db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Where("control_stream_id = ?", id).Delete(&domains.Command{}).Error; err != nil {
            return err
        }
        return tx.Delete(&domains.ControlStream{}, "id = ?", id).Error
    })
}
```

Identical pattern, identical defect: no `system_controlstreams` cleanup before
`tx.Delete(&domains.ControlStream{}, ...)`. FK
`fk_system_controlstreams_control_stream` will trip identically.

### `SystemRepository.deleteCascade` — `internal/repository/system_repository.go:189-235`

```go
// deleteSystemDatastreams plucks the per-system datastream IDs, deletes
// observations, then deletes the datastreams. Does NOT clean system_datastreams.
func (r *SystemRepository) deleteSystemDatastreams(tx *gorm.DB, systemID string) error {
    var datastreamIDs []string
    tx.Model(&domains.Datastream{}).Where("system_id = ?", systemID).Pluck("id", &datastreamIDs)
    if len(datastreamIDs) > 0 {
        tx.Where("datastream_id IN ?", datastreamIDs).Delete(&domains.Observation{})
    }
    return tx.Where("system_id = ?", systemID).Delete(&domains.Datastream{}).Error // ← trips fk_system_datastreams_datastream
}
```

Repo-wide search confirms the asymmetry:

| Cleanup | Present in `deleteCascade`? |
|---|---|
| `DELETE FROM system_deployments WHERE system_id = ?` | yes (line 226) |
| `DELETE FROM system_procedures WHERE system_id = ?` | yes (line 229) |
| `DELETE FROM system_datastreams WHERE system_id = ?` | **no** (only joins on read at lines 443/452) |
| `DELETE FROM system_controlstreams WHERE system_id = ?` | **no** (only joins on read at lines 452) |

`git grep -n "system_datastreams\|system_controlstreams" -- internal/repository/system_repository.go`
returns only the four read-path JOIN clauses.

## Bug 2 — `DeploymentRepository.Delete` has no cascade

### Repository signature — `internal/repository/deployment_repository.go:111-113`

```go
func (r *DeploymentRepository) Delete(id string) error {
    return r.db.Delete(&domains.Deployment{}, "id = ?", id).Error
}
```

### Handler — `internal/api/deployment_handler.go:118-129`

```go
func (h *DeploymentHandler) DeleteDeployment(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "id")
    if err := h.repo.Delete(id); err != nil { ... 500 ... }
    w.WriteHeader(http.StatusNoContent)
}
```

No `r.URL.Query().Get("cascade")` parsing. No bool param to repository. No
escape hatch for clients when the deployment has been linked into
`system_deployments`. FK `fk_system_deployments_deployment` will trip on any
linked deployment.

### Repo-wide DELETE inventory

`git grep -n "func .r .\w+Repository.\? Delete(" -- internal/repository/`:

| Repository | Cascade param? |
|---|---|
| `SystemRepository.Delete(id string, cascade bool)` | yes |
| `DatastreamRepository.Delete(id string, cascade bool)` | yes |
| `ControlStreamRepository.Delete(id string, cascade bool)` | yes |
| `DeploymentRepository.Delete(id string)` | **no** |
| `CommandRepository.Delete(id string)` | no |
| `FeatureRepository.Delete(id string)` | no |
| `ObservationRepository.Delete(id string)` | no |
| `ProcedureRepository.Delete(id string)` | no |
| `PropertyRepository.Delete(id string)` | no |
| `SamplingFeatureRepository.Delete(id string)` | no |
| `SystemEventRepository.Delete(systemID, eventID string)` | no |
| `SystemHistoryRepository.Delete(systemID, revID string)` | no |

3 of 12 repositories support cascade. Exactly the count claimed.

## Bug 3 — Silent orphaning: natural 1:N children have no FK

### Domain-side relationship-tag inventory

`git grep -n "foreignKey:\|references:" -- internal/model/domains/`:

```
internal/model/domains/sampling_feature.go:40:  // defined on the System side ...
internal/model/domains/sampling_feature.go:42:  //ParentSystem *System `gorm:"->;references:ID" json:"parentSystem,omitempty"`  // commented out
internal/model/domains/system.go:62:    SystemKind Procedure `gorm:"foreignKey:SystemKindID;" json:"-"`
internal/model/domains/system.go:67:    SamplingFeatures []SamplingFeature `gorm:"foreignKey:ParentSystemID;"`
```

**Only two** active `foreignKey:` tags exist in the entire domain package.
Everything else relies on m2m `gorm:"many2many:..."` tags, which emit FKs only
on the join table (the m2m FKs we see in the FK list).

### FK list (production + HEAD; 14 user FKs + 1 PostGIS internal `layer.topology_id_fkey`)

Verbatim from `docs/research/evidence/issue-002/fk-constraints-head-2026-04-30.txt`:

```
sampling_features.fk_systems_sampling_features (parent_system_id → systems.id)
systems.fk_systems_system_kind                 (system_kind_id  → procedures.id)
system_datastreams.fk_..._datastream | _system
system_controlstreams.fk_..._control_stream | _system
system_deployments.fk_..._deployment | _system
system_procedures.fk_..._procedure | _system
procedure_observed_properties.fk_..._procedure | _property
procedure_controlled_properties.fk_..._procedure | _property
```

### Confirmed FK absences on natural 1:N children

| Column | FK present? | Domain-side tag declared? |
|---|---|---|
| `observations.datastream_id` | no | no `Observations []Observation` field on `Datastream` |
| `commands.control_stream_id` | no | no `Commands []Command` field on `ControlStream` |
| `datastreams.system_id` | no | none |
| `system_events.system_id` | no | none |
| `system_history_revisions.system_id` | no | none |
| `deployments.parent_deployment_id` | no | none (closure managed by trigger) |
| `systems.parent_system_id` | no FK on `systems` (only the `sampling_features` projection) | none |

GORM's `AutoMigrate` only emits the `REFERENCES` clause when a Go-level
relationship tag is declared. Without it the column exists but is unconstrained.
This matches the FK inventory exactly.

### Cascade path absences on the natural side

`SystemRepository.deleteCascade` does NOT clean `SystemEvent`. `git grep` for
`SystemEvent` or `system_events` in `internal/repository/system_repository.go`
returns zero matches. Once Bug 1 is fixed and the system delete actually
succeeds, `system_events` rows would be silently orphaned.

### Why "silent orphaning" is currently masked

Because every datastream is created via `POST /systems/{id}/datastreams`
(`internal/api/router.go:126` is the only POST route), every datastream has a
`system_datastreams` row, so the m2m FK fires before the natural-relation
orphaning can occur — Bug 1 *masks* Bug 3. The moment Bug 1 is fixed without
also addressing Bug 3, the deletion succeeds and observations/commands/events
silently orphan. This is the issue's "worst possible combination" framing,
correctly captured.

## Conclusion

All three structural claims in the issue body are correct, with the line
numbers, error class names, and code patterns matching exactly. The only
trivial drift is that the issue references HEAD `4b99421`; `git log` confirms
no relevant code changes through HEAD `edc8459`.
