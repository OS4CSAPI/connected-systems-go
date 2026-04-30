# Issue #2 — DELETE on resources with related child rows returns HTTP 500 from FK violation

- **URL:** <https://github.com/OS4CSAPI/connected-systems-go/issues/2>
- **State at evaluation:** open
- **Labels:** `bug`
- **Filed by:** Sam-Bolling
- **Filed:** 2026-04-17
- **Evaluated against cs-go HEAD:** `4b994212d6f09f522a39e77a712c16aeab96b3e0` (parallel HEAD stack), with cross-check against the production stack pre-`1562201`.
- **Date of evaluation:** 2026-04-30
- **Evaluation methodology:** static code review of every `DELETE` route + handler + repository at HEAD; PostgreSQL FK introspection on **both** the production DB and the parallel HEAD DB (via `pg_constraint`); live empirical T1–T15 against the parallel HEAD deployment exercising `DELETE` with and without `?cascade=true`, with body capture, status capture, and server-side log inspection (zap stack traces, GORM SQL traces); cross-handler equality check across all 12 `r.Delete("/", ...)` routes.
- **Raw evidence:** [`docs/research/evidence/issue-002/`](../evidence/issue-002/)
  - [`fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt) — definitive FK list, HEAD DB
  - [`fk-constraints-prod-2026-04-30.txt`](../evidence/issue-002/fk-constraints-prod-2026-04-30.txt) — definitive FK list, production DB
  - [`delete-tests-head-2026-04-30.txt`](../evidence/issue-002/delete-tests-head-2026-04-30.txt) — T1–T15 transcripts
  - [`server-error-logs-2026-04-30.txt`](../evidence/issue-002/server-error-logs-2026-04-30.txt) — zap error logs with GORM stack traces
  - [`static-analysis-source-2026-04-30.txt`](../evidence/issue-002/static-analysis-source-2026-04-30.txt) — handler/repository source extracts

---

## Sources Consulted

### Primary (authoritative)

- [`internal/api/router.go`](../../../internal/api/router.go) at HEAD — every `r.Delete(...)` route across the 12 resource subrouters.
- [`internal/api/datastream_handler.go`](../../../internal/api/datastream_handler.go) `DeleteDatastream` (lines 186–195) — representative cascade-aware handler.
- [`internal/api/system_handler.go`](../../../internal/api/system_handler.go) `DeleteSystem` (lines 151–162) — cascade-aware handler.
- [`internal/api/control_stream_handler.go`](../../../internal/api/control_stream_handler.go) `DeleteControlStream` (lines 197–208) — cascade-aware handler.
- [`internal/api/deployment_handler.go`](../../../internal/api/deployment_handler.go) `DeleteDeployment` (lines 118–129) — non-cascade handler.
- [`internal/repository/datastream_repository.go`](../../../internal/repository/datastream_repository.go) `Delete` (lines 134–146) — cascade implementation that does not clear m2m join.
- [`internal/repository/system_repository.go`](../../../internal/repository/system_repository.go) `Delete` + `deleteCascade` (lines 184–266).
- [`internal/repository/control_stream_repository.go`](../../../internal/repository/control_stream_repository.go) `Delete`.
- [`internal/repository/deployment_repository.go`](../../../internal/repository/deployment_repository.go) `Delete` — has no cascade parameter at all.
- [`internal/model/domains/datastream.go`](../../../internal/model/domains/datastream.go) line 57 — `Systems []System \`gorm:"many2many:system_datastreams;"\`` — the GORM tag that auto-creates the FK that fires the 500.
- [`cmd/server/main.go`](../../../cmd/server/main.go) line 44 — `gorm.Open(postgres.Open(dsn), &gorm.Config{})` (no `DisableForeignKeyConstraintWhenMigrating`).
- **Live HEAD deployment** at `https://129-80-248-53.sslip.io/csapi-go-head/` and **production deployment** at `https://129-80-248-53.sslip.io/csapi-go/` — both probed 2026-04-30.
- **Production DB** `connected-systems-go-db-1` and **HEAD DB** `csapi-head-db-1` — `psql` against `connected_systems` schema.

### Supporting

- Issues [#13](https://github.com/OS4CSAPI/connected-systems-go/issues/13), [#14](https://github.com/OS4CSAPI/connected-systems-go/issues/14), [#15](https://github.com/OS4CSAPI/connected-systems-go/issues/15) (filed during issue #1 evaluation) — establish the parallel-HEAD methodology and the m2m FK side-effect surface.
- [`docs/research/issue-evaluations/issue-001.md`](issue-001.md) §2.7 — the same production deployment is verified to predate commit `1562201`.

---

## 1. Issue body — load-bearing claims

| # | Claim | Source in issue body |
|---|---|---|
| C1 | DELETE requests on resources with referencing child rows return HTTP 500. | "Problem Statement", "Actual behavior" |
| C2 | The 500 is caused by a PostgreSQL foreign-key constraint violation (SQLSTATE 23503). | "Actual behavior", "Root Cause" |
| C3 | The raw PostgreSQL error message ("violates foreign key constraint…") is exposed in the API response body. | "Actual behavior" excerpt |
| C4 | There is no `?cascade=true` (or equivalent) option on DELETE endpoints to allow ordered teardown. | "Recommended fix" implies absence |
| C5 | The bug blocks `--clean` teardown in `ogc-csapi-explorer` and `OSHConnect-Python` publisher integration tests, leaving stranded resources. | "Impact" |
| C6 (implicit) | The FK violations originate from declared parent→child FK constraints (e.g. `observations.datastream_id` → `datastreams.id`). | Implied by phrase "FK on parent→child resource relationships" |

Stated severity: P1-Critical · Stated category: API Behavior / Error Handling · Stated ownership: connected-systems-go.

---

## 2. Verification

### 2.1 — All 12 DELETE handlers at HEAD: same generic 500-on-error pattern (supports C1)

**Evidence — every `r.Delete("/", ...)` route in `internal/api/router.go`:** lines 96, 108, 132, 138, 155, 172, 189, 200, 212, 228, 239, 251 (feature, system, system_event, history, datastream, controlstream, command, observation, deployment, procedure, sampling_feature, property).

**Evidence — `DeleteDatastream` (representative, [`datastream_handler.go`](../../../internal/api/datastream_handler.go#L186-L195)):**

```go
func (h *DatastreamHandler) DeleteDatastream(w http.ResponseWriter, r *http.Request) {
    id := chi.URLParam(r, "dataStreamId")
    cascade := r.URL.Query().Get("cascade") == "true"
    if err := h.repo.Delete(id, cascade); err != nil {
        h.logger.Error("Failed to delete datastream", zap.String("id", id), zap.Error(err))
        render.Status(r, http.StatusInternalServerError)
        render.JSON(w, r, map[string]string{"error": "Failed to delete datastream"})
        return
    }
    w.WriteHeader(http.StatusNoContent)
}
```

**Analysis:** Every DELETE handler has the same shape: any non-nil repository error → `render.Status(..., 500)` + a fixed `{"error": "Failed to delete X"}` body. There is **zero error-class discrimination**: no `errors.As(*pgconn.PgError)`, no SQLSTATE inspection, no 404 path for not-found, no 409 for FK conflicts. C1 is structurally true: any error of any kind, including FK violation, becomes 500. (See §2.4 for the 404 follow-up surface.)

### 2.2 — `?cascade=true` IS exposed on 3 of 12 endpoints (refines C4)

**Evidence — `grep -rn "cascade" internal/api/`:**

```
internal/api/datastream_handler.go:188:   cascade := r.URL.Query().Get("cascade") == "true"
internal/api/control_stream_handler.go:199: cascade := r.URL.Query().Get("cascade") == "true"
internal/api/system_handler.go:153:        cascade := r.URL.Query().Get("cascade") == "true"
```

**Analysis:** Three handlers (`Datastream`, `ControlStream`, `System`) accept `?cascade=true`. The other nine (`feature`, `system_event`, `history`, `command`, `observation`, `deployment`, `procedure`, `sampling_feature`, `property`) have no cascade option. The issue body is **incorrect that no cascade option exists** (refines C4) — but C4's spirit is preserved: the cascade option is undocumented (not in any OpenAPI stub — see §2.6) and, as §2.5 demonstrates empirically, **`cascade=true` itself returns 500** in the most common case. So a partial mitigation exists in code but is *non-functional* at runtime.

### 2.3 — Real DB FK constraints: only on GORM-generated m2m join tables, NOT on natural parent→child relations (refutes C6)

**Evidence — definitive FK list, HEAD DB ([fk-constraints-head-2026-04-30.txt](../evidence/issue-002/fk-constraints-head-2026-04-30.txt), excerpt):**

```
table                            | conname                                       | def
---------------------------------|-----------------------------------------------|----------------------------------------------------
sampling_features                | fk_systems_sampling_features                  | FOREIGN KEY (parent_system_id) REFERENCES systems(id)
system_controlstreams            | fk_system_controlstreams_control_stream       | FOREIGN KEY (control_stream_id) REFERENCES control_streams(id)
system_controlstreams            | fk_system_controlstreams_system               | FOREIGN KEY (system_id) REFERENCES systems(id)
system_datastreams               | fk_system_datastreams_datastream              | FOREIGN KEY (datastream_id) REFERENCES datastreams(id)
system_datastreams               | fk_system_datastreams_system                  | FOREIGN KEY (system_id) REFERENCES systems(id)
system_deployments               | fk_system_deployments_deployment              | FOREIGN KEY (deployment_id) REFERENCES deployments(id)
system_deployments               | fk_system_deployments_system                  | FOREIGN KEY (system_id) REFERENCES systems(id)
system_procedures                | fk_system_procedures_procedure                | FOREIGN KEY (procedure_id) REFERENCES procedures(id)
system_procedures                | fk_system_procedures_system                   | FOREIGN KEY (system_id) REFERENCES systems(id)
procedure_controlled_properties  | fk_procedure_controlled_properties_property   | FOREIGN KEY (property_id) REFERENCES properties(id)
procedure_observed_properties    | fk_procedure_observed_properties_property     | FOREIGN KEY (property_id) REFERENCES properties(id)
systems                          | fk_systems_system_kind                        | FOREIGN KEY (system_kind_id) REFERENCES procedures(id)
```

The production DB is **identical** ([fk-constraints-prod-2026-04-30.txt](../evidence/issue-002/fk-constraints-prod-2026-04-30.txt) — same 14 rows, same definitions).

**Crucial absences:** `observations.datastream_id` has **no FK**; `commands.control_stream_id` has **no FK**; `datastreams.system_id` has **no FK**; `system_events.system_id` has **no FK**; `system_history_revisions.system_id` has **no FK**; `deployments.parent_deployment_id` and `systems.parent_system_id` (other than `sampling_features`) have **no FK** (closure tables are managed by triggers — see [`internal/repository/closure.go`](../../../internal/repository/closure.go)).

**Source of the FKs:** GORM `AutoMigrate` reads the `many2many:system_datastreams` tag on [`datastream.go:57`](../../../internal/model/domains/datastream.go#L57) and similar tags on `controlstream.go`, `deployment.go`, `procedure.go`. For each m2m it auto-creates a join table with two RESTRICT FKs (one to each side). No such tags exist on the natural 1:N relations (e.g. there is no `Observations []Observation \`gorm:"foreignKey:DatastreamID;references:ID"\`` field) — so those columns get **no FK**.

**Analysis:** This refutes C6's implicit model. The issue's mental model is "DELETE blocked by parent→child FK". Reality is the inverse: there is **no FK at all on the natural parent→child columns**, so deleting an *empty* datastream succeeds **even when child observations still exist** — silently orphaning them in `observations.datastream_id`. The 500 fires only when the resource has been **linked into a many2many join**, i.e. a system→datastream / system→controlstream / system→deployment / system→procedure relationship. The system has Schrödinger's referential integrity: half the relations are RESTRICTed (so DELETE 500s loudly) and half have no constraint at all (so DELETE silently orphans).

### 2.4 — Sanitized error body, not raw PG error (refutes C3)

**Evidence — actual API response body, T1 (default DELETE on a datastream linked to a system, HEAD-`4b99421` / GORM logs):**

```
< HTTP/2 500
< content-type: application/json
{"error":"Failed to delete datastream"}
```

(40 bytes; transcript [delete-tests-head-2026-04-30.txt](../evidence/issue-002/delete-tests-head-2026-04-30.txt) lines for T5 and T8.)

**Evidence — server-side zap log for the same request ([server-error-logs-2026-04-30.txt](../evidence/issue-002/server-error-logs-2026-04-30.txt)):**

```
2026/04/30 18:06:22 /app/internal/repository/datastream_repository.go:136 ERROR: update or delete on table
"datastreams" violates foreign key constraint "fk_system_datastreams_datastream" on table
"system_datastreams" (SQLSTATE 23503)
[1.380ms] [rows:0] DELETE FROM "datastreams" WHERE id = '8fb3e570-…'
{"level":"error","ts":1777572382.4367697,"caller":"api/datastream_handler.go:190","msg":"Failed to delete
datastream","id":"8fb3e570-…","error":"ERROR: update or delete on table \"datastreams\" violates foreign
key constraint \"fk_system_datastreams_datastream\" on table \"system_datastreams\" (SQLSTATE 23503)",
"stacktrace": "[…full Go stack…]"}
```

**Analysis:** The raw PostgreSQL error is logged at server-side ERROR level (with full stack trace), but the response body is a sanitized constant `{"error":"Failed to delete X"}`. The issue's quoted excerpt "violates foreign key constraint" came from **server logs**, not the response body. C3 is **false** for cs-go HEAD. (This is a security positive — no DB schema leak — but a UX negative: the client receives no actionable diagnosis. See §6 for the recommended remediation.)

### 2.5 — `?cascade=true` is **also broken** for any datastream/system that has been linked to a system (new finding, not in issue body)

**Evidence — repository cascade implementations:**

`datastream_repository.go` lines 134–146:

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

`system_repository.go` `deleteSystemDatastreams` (called from `deleteCascade`):

```go
func (r *SystemRepository) deleteSystemDatastreams(tx *gorm.DB, systemID string) error {
    var datastreamIDs []string
    if err := tx.Model(&domains.Datastream{}).Where("system_id = ?", systemID).Pluck("id", &datastreamIDs).Error; err != nil {
        return err
    }
    if len(datastreamIDs) > 0 {
        if err := tx.Where("datastream_id IN ?", datastreamIDs).Delete(&domains.Observation{}).Error; err != nil {
            return err
        }
    }
    return tx.Where("system_id = ?", systemID).Delete(&domains.Datastream{}).Error  // ← this fires fk_system_datastreams_datastream
}
```

**The cascade bug:** GORM's plain `db.Delete(&domains.Datastream{}, …)` does *not* automatically clear the `system_datastreams` m2m join table. (Clearing requires `db.Select(clause.Associations).Delete(...)` or an explicit `tx.Exec("DELETE FROM system_datastreams WHERE datastream_id = ?")`.) Both `DatastreamRepository.Delete(cascade=true)` and `SystemRepository.deleteSystemDatastreams` issue raw `tx.Delete(&Datastream{}, …)` against records that still have join rows pointing at them, immediately tripping the RESTRICT FK.

`system_repository.go` deleteCascade *does* explicitly clean some join tables:

```go
if err := tx.Exec("DELETE FROM system_deployments WHERE system_id = ?", systemID).Error; err != nil { return err }
if err := tx.Exec("DELETE FROM system_procedures WHERE system_id = ?", systemID).Error; err != nil { return err }
return tx.Delete(&domains.System{}, "id = ?", systemID).Error
```

…but **not** `system_datastreams` and **not** `system_controlstreams` — the very joins that `deleteSystemDatastreams` and `deleteSystemControlStreams` will collide with.

**Live empirical confirmation:**

| Test | Request | Status | Notes |
|------|---------|--------|-------|
| T8 | `DELETE /datastreams/{id}?cascade=true` (datastream has 1 system join + 1 observation) | **500** | Server log: `fk_system_datastreams_datastream` |
| T11 | `DELETE /systems/{id}?cascade=true` (system has 1 datastream) | **500** | Server log: `fk_system_datastreams_system` (raised by `system_repository.go:186`) |
| T13 | `DELETE /deployments/{id}` (no children, no joins) | 204 | OK — confirms 500 is FK-driven, not blanket |
| T15 | `DELETE /procedures/{id}` (no children, no joins) | 204 | OK |

**Analysis:** This is **the most damaging finding in the evaluation**, and it is *not in the issue body*. The single mitigation that the cs-go codebase exposes for the bug — `?cascade=true` — is itself broken for the canonical case (datastream or system linked to a system, which is the only way to *create* a datastream via the API, see route in `router.go` line 142: `POST /systems/{id}/datastreams`). A client cannot work around the bug from the application layer because no API endpoint exposes the m2m join-table for direct manipulation.

### 2.6 — `DeploymentRepository.Delete` has **no** cascade parameter at all (refines C4)

**Evidence — [`deployment_repository.go`](../../../internal/repository/deployment_repository.go) line 111:**

```go
func (r *DeploymentRepository) Delete(id string) error {
    return r.db.Delete(&domains.Deployment{}, "id = ?", id).Error
}
```

And [`deployment_handler.go`](../../../internal/api/deployment_handler.go) line 118 calls it as `h.repo.Delete(id)` — no cascade flag is read, and no transactional cleanup of `system_deployments` join rows is performed.

**Analysis:** Empirically, T13 succeeded because the deployment had no `system_deployments` rows (no system was added). Any deployment that has been linked to a system via `POST /deployments/{id}/systems` (or has a sub-deployment via the closure trigger) will collide with `fk_system_deployments_deployment` at delete. Same hidden-cascade-broken pattern as Datastream/System, except even more thoroughly broken because there is no escape hatch in the API surface at all.

### 2.7 — Cross-DB consistency: bug is identical on production (corroborates C5)

The production DB at `129-80-248-53.sslip.io/csapi-go/` and the HEAD DB at `…/csapi-go-head/` have **byte-identical FK constraint sets**. The same `Datastream.Systems []System \`gorm:"many2many:system_datastreams;"\`` field exists at the pre-`1562201` production tree (verified during the issue #1 evaluation, see issue-001.md §2.7), and the same handler/repository code exists too. Any client (`ogc-csapi-explorer`, `OSHConnect-Python`) hitting either deployment will see the same 500 on `DELETE /datastreams/{id}` and `DELETE /systems/{id}` once a datastream has been associated with a system.

C5 is corroborated structurally: the bug is reproducible in the exact production environment those clients run against.

---

## 3. Reproduction summary (live, HEAD-`4b99421`)

Bootstrap (T1–T4):

```bash
SYS_ID=$(curl -sS -i -X POST "$BASE/systems" -H "Content-Type: application/geo+json" \
  -d '{"type":"Feature","properties":{"uid":"…","name":"…","featureType":"http://www.w3.org/ns/sosa/Sensor"},"geometry":null}' \
  | grep -i '^location:' | awk '{print $2}' | tr -d '\r' | sed 's|.*/||')
DS_ID=$(curl -sS -i -X POST "$BASE/systems/$SYS_ID/datastreams" -H "Content-Type: application/json" \
  -d '{"name":"DS","observedProperties":[{"definition":"http://x","label":"X"}], …}' \
  | grep -i '^location:' | awk '{print $2}' | tr -d '\r' | sed 's|.*/||')
curl -sS -X POST "$BASE/datastreams/$DS_ID/observations" -H "Content-Type: application/om+json" \
  -d '{"phenomenonTime":"2026-04-30T00:00:00Z","resultTime":"2026-04-30T00:00:00Z","result":42}'
```

Symptom (T5 / T8 / T10 / T11):

```bash
curl -i -X DELETE "$BASE/datastreams/$DS_ID"                 # 500, body 40B
curl -i -X DELETE "$BASE/datastreams/$DS_ID?cascade=true"    # 500, body 40B   ← cascade also broken
curl -i -X DELETE "$BASE/systems/$SYS_ID"                    # 500
curl -i -X DELETE "$BASE/systems/$SYS_ID?cascade=true"       # 500             ← cascade also broken
```

Negative control (T13 / T15):

```bash
DEP_ID=$(curl -sS -i -X POST "$BASE/deployments" … | …); curl -i -X DELETE "$BASE/deployments/$DEP_ID"  # 204
PROC_ID=$(curl -sS -i -X POST "$BASE/procedures" … | …); curl -i -X DELETE "$BASE/procedures/$PROC_ID"  # 204
```

Full transcripts in [`docs/research/evidence/issue-002/delete-tests-head-2026-04-30.txt`](../evidence/issue-002/delete-tests-head-2026-04-30.txt).

---

## 4. Verdict on each load-bearing claim

| # | Claim | Verdict | Notes |
|---|---|---|---|
| C1 | DELETE returns 500 when child rows exist. | **Confirmed** | All 12 handlers map any error to 500. T5/T10 reproduce. |
| C2 | Caused by FK violation (SQLSTATE 23503). | **Confirmed** | Server log shows verbatim `(SQLSTATE 23503)` for both datastream and system deletes. |
| C3 | Raw PG error is exposed in API response. | **Refuted** | Response body is `{"error":"Failed to delete X"}` (40 B). Raw PG error is in server logs only. |
| C4 | No cascade option exists. | **Partially refuted, materially confirmed** | Datastream / ControlStream / System handlers expose `?cascade=true`, but it's empirically broken (§2.5) — so the user-facing observation "cannot cascade delete" is correct in effect. Deployment / procedure / 7 others have no cascade param at all. |
| C5 | Blocks `--clean` teardown of integration suites. | **Confirmed structurally** | Same bug on prod & HEAD; the canonical create-then-tear-down workflow goes through `POST /systems/{id}/datastreams` which always creates the join row, so DELETE of either side will always collide. |
| C6 | FK violations come from natural parent→child FKs. | **Refuted** | The natural 1:N relations have **no FK at all**. The colliding FKs are on the GORM-generated m2m join tables (`fk_system_datastreams_datastream`, `fk_system_datastreams_system`, etc.). This inverts the issue's mental model. |

**Net:** The reported user-visible symptom (DELETE → 500, blocks teardown) is fully confirmed and reproducible. The internal causal model in the issue body is partially wrong: the FKs are on m2m join tables (not natural parent→child), the existing `?cascade=true` mitigation is broken, the response body does not leak raw SQL, and `Deployment.Delete` lacks even a cascade hook. Additionally, **a silent data-integrity hazard exists in the opposite direction**: deleting a datastream that has *no* system join but *does* have observations would succeed and orphan its observations (because there is no FK from `observations.datastream_id`).

---

## 5. Reasoning summary

cs-go uses GORM with default config (`&gorm.Config{}`). At AutoMigrate time, GORM walks the struct fields and emits one auto-FK per `many2many` tag, RESTRICT-by-default — but emits no FK for plain `varchar` foreign-key columns that lack a Go-level relationship tag. The result is a half-implemented referential integrity model where DELETE 500s loudly when the m2m side is non-empty and silently orphans when only the un-FK'd 1:N side is non-empty.

The handler layer is uniformly thin: `repo.Delete(...)` errors map 1:1 to HTTP 500 with a static sanitized message. The error logged by zap *is* useful (full SQLSTATE + GORM call site), so this is purely a remote-API contract problem, not an observability problem.

The cascade implementations look correct on inspection but rely on `gorm.DB.Delete` propagating to many2many join rows — which it does not. The implementations would work if either (a) the join-table FK declared `ON DELETE CASCADE` (it does not — see §2.3 "RESTRICT-by-default"), or (b) the cascade code explicitly issued `DELETE FROM <join_table> WHERE … = ?` before deleting the resource (`SystemRepository.deleteCascade` does this for `system_deployments` and `system_procedures` but forgets `system_datastreams` and `system_controlstreams`).

---

## 6. Recommendation (for upstream remediation)

Three coordinated fixes:

1. **Repository layer — make cascade work.** In `DatastreamRepository.Delete` and the cascade path of `SystemRepository.deleteCascade`/`deleteSystemDatastreams`/`deleteSystemControlStreams`, explicitly clear the relevant join tables before deleting the resource. Mirror the pattern already used for `system_deployments`/`system_procedures`:

    ```go
    if err := tx.Exec("DELETE FROM system_datastreams WHERE datastream_id = ?", id).Error; err != nil { return err }
    return tx.Delete(&domains.Datastream{}, "id = ?", id).Error
    ```

   Apply symmetrically for the `system_id` direction in `SystemRepository.deleteCascade` and for `system_controlstreams` and `procedure_*_properties` join tables.

2. **Handler layer — map error classes to HTTP codes.** Add a small helper:

    ```go
    func mapDeleteError(err error) (int, string) {
        var pgErr *pgconn.PgError
        if errors.As(err, &pgErr) && pgErr.Code == "23503" {
            return http.StatusConflict, "Resource has dependent rows; use ?cascade=true or remove dependents first"
        }
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return http.StatusNotFound, "Resource not found"
        }
        return http.StatusInternalServerError, "Internal error"
    }
    ```

   Apply to all 12 DELETE handlers. Keeps the response body sanitized (no raw SQL leak) but gives clients an actionable status code.

3. **Either close the data-integrity gap on the natural 1:N side, or document the orphaning behaviour.** Adding GORM `\`gorm:"foreignKey:DatastreamID;references:ID;constraint:OnDelete:RESTRICT\`` tags to `Observation.DatastreamID` etc. is the schema-correct fix; if that's deemed too disruptive, the cascade implementation should at least delete the children explicitly even when no FK enforces it.

Optional: add `cascade=true` support to `DeploymentRepository.Delete` (§2.6).

---

## 7. Open questions / follow-ups

- **Should `cascade=true` be the default for top-level resources?** The OGC API – Connected Systems spec (Part 1 §7.5, Part 2 §9.x) does not prescribe DELETE cascade semantics. Reasonable choices range from "always cascade" (per-Connected Systems node ownership semantics imply hierarchical lifetime) to "never cascade, force explicit ordered teardown". A defaulting decision is upstream policy.
- **Are observations meant to be reified resources independent of their datastream?** If yes, deleting a datastream should *not* cascade to observations (and the FK from `observations.datastream_id` should be `ON DELETE SET NULL`); if no, the FK should be `ON DELETE CASCADE`. The current state — no FK + no cascade — is the worst of both.
- **Is the m2m model the right data model?** Issue [#15](https://github.com/OS4CSAPI/connected-systems-go/issues/15) already raised this for the GET-list "shared datastream" leak. The same `Datastream.Systems []System \`gorm:"many2many"\`` field is the upstream cause of *both* issues #15 and the present #2. A schema redesign that removes the m2m and uses a single `system_id` FK with proper RESTRICT/CASCADE would fix both — but it's a breaking schema migration and warrants its own design discussion.

---

## 8. Follow-up surfaces uncovered by this evaluation

These are candidates for new issues, NOT yet filed (per phase-7 process — review at end of pass before deciding which to file):

1. **`?cascade=true` is non-functional on Datastream and System** (§2.5). Severity P1: the only documented mitigation in code does not work. Title candidate: `[bug] DELETE /datastreams/{id}?cascade=true returns 500 — cascade impl does not clear system_datastreams m2m join`.
2. **Generic 500 mapping for all DB errors in 12 DELETE handlers** (§2.1, §2.4). Severity P2 (UX/observability). Title candidate: `[enhancement] DELETE handlers should map FK violation (SQLSTATE 23503) to 409 and missing rows to 404`.
3. **`DeploymentRepository.Delete` has no cascade parameter** (§2.6). Severity P2. Title candidate: `[bug] DELETE /deployments/{id} cannot remove a deployment that has been linked to a system`.
4. **Silent orphaning of observations when datastream has no m2m join** (§2.3). Severity P2 (data-integrity). Title candidate: `[bug] DELETE /datastreams/{id} silently orphans child observations when datastream has no system association (no FK on observations.datastream_id)`.
5. **m2m join-FK side-effect on commands too** — by symmetry with §2.3, deleting a control_stream that's linked to a system will hit `fk_system_controlstreams_control_stream` and the cascade impl in `control_stream_repository.go` has the same shape (verified statically in `static-analysis-source-2026-04-30.txt`). Could fold into #1 above.

A single combined umbrella issue covering #1+#3+#5 ("DELETE cascade is broken across resource types — m2m join FKs not cleaned") may be more actionable than three separate issues.
