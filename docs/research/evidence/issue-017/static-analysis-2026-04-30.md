# Issue #17 — Static analysis (HEAD `0255a10`)

Date: 2026-04-30. Issue created in same session as HEAD push; verified
`git log` shows no relevant code drift on referenced files.

## Inventory: 12 DELETE routes, 12 handlers

`internal/api/router.go`:

```
:96   r.Delete("/", featureHandler.DeleteFeature)
:108  r.Delete("/", systemHandler.DeleteSystem)
:132  r.Delete("/", systemEventHandler.DeleteEventByID)
:138  r.Delete("/", systemHandler.DeleteSystemHistoryRevision)
:155  r.Delete("/", datastreamHandler.DeleteDatastream)
:172  r.Delete("/", controlStreamHandler.DeleteControlStream)
:189  r.Delete("/", commandHandler.DeleteCommand)
:200  r.Delete("/", observationHandler.DeleteObservation)
:212  r.Delete("/", deploymentHandler.DeleteDeployment)
:228  r.Delete("/", procedureHandler.DeleteProcedure)
:239  r.Delete("/", samplingFeatureHandler.DeleteSamplingFeature)
:251  r.Delete("/", propertyHandler.DeleteProperty)
```

12 routes, all wired to `<X>Handler.Delete<X>`. Count matches issue claim.

## Per-handler pre-flight 404 audit — ISSUE BODY OVERSTATES

| Handler | File:line | Pre-flight `GetByID` 404? |
|---|---|---|
| `DeleteCommand` | `command_handler.go:179` | **YES** — `if _, err := h.repo.GetByID(id); err != nil { 404 }` at line 182-186 |
| `DeleteControlStream` | `control_stream_handler.go:197` | no |
| `DeleteDatastream` | `datastream_handler.go:186` | no |
| `DeleteDeployment` | `deployment_handler.go:118` | no |
| `DeleteFeature` | `feature_handler.go:191` | **YES** — `_, err := h.repo.GetByCollectionAndID(...); if err != nil { 404 }` at line 199-205 |
| `DeleteObservation` | `observation_handler.go:152` | **YES** — `if _, err := h.repo.GetByID(id); err != nil { 404 }` at line 155-159 |
| `DeleteProcedure` | `procedure_handler.go:120` | no |
| `DeleteProperty` | `property_handler.go:122` | no |
| `DeleteSamplingFeature` | `sampling_feature_handler.go:159` | no |
| `DeleteEventByID` | `system_event_handler.go:230` | **YES** — `if _, err := h.repo.GetByID(systemID, eventID); err != nil { 404 }` at line 238-241 |
| `DeleteSystem` | `system_handler.go:151` | no |
| `DeleteSystemHistoryRevision` | `system_history_handler.go:138` | **YES** — `if _, err := h.historyRepo.GetByID(systemID, revID); err != nil { 404 }` at line 146-149 |

**5 of 12** handlers (Command, Feature, Observation, SystemEvent, SystemHistory)
already implement pre-flight 404 via a separate `GetByID` call. The issue body's
framing — "no 404 path for missing resources" — is **too strong**.

The remaining 7 handlers (ControlStream, Datastream, Deployment, Procedure,
Property, SamplingFeature, System) have no pre-flight check and rely entirely
on the result of `repo.Delete(...)`.

## Repository `Delete` semantics — GORM returns nil for non-existent IDs

GORM's `db.Delete(&model{}, "id = ?", id)` returns:

- `nil` error when 0 rows match (success-no-op semantics)
- non-nil error only on actual DB-layer failure (FK violation, connection loss, etc.)

This is why the 7 no-preflight handlers return **204** for non-existent IDs
(see live-test results), not 500 — directly contradicting the issue body's
claim that "missing resource → 500 indistinguishably".

The issue body is correct that the 500-collapse problem exists for actual
errors (FK conflicts), but wrong about the 404 channel. The accurate
characterization is:

| Failure mode on `Delete` | Current behavior | Issue body claim |
|---|---|---|
| Missing ID, 5 handlers with preflight | 404 | "all 12 → 500" — **wrong** |
| Missing ID, 7 handlers w/o preflight | 204 (silent success) | "all 12 → 500" — **wrong** |
| FK constraint violation (23503) | 500 + opaque body | 500 — correct |
| Unique constraint violation (23505) on DELETE | n/a (DELETE doesn't unique-violate) | partly correct in scope-creep on POST |
| Other PG error | 500 + opaque body | 500 — correct |

## Error-class discrimination — fully absent for actual errors

`git grep "errors\.As\|pgconn\|SQLSTATE\|ErrRecordNotFound\|StatusConflict" -- internal/api/`
returns **zero** matches. Confirmed:

- No handler imports `github.com/jackc/pgx/v5/pgconn`.
- No handler calls `errors.As(err, &pgErr)`.
- No handler uses `errors.Is(err, gorm.ErrRecordNotFound)`.
- No handler returns `http.StatusConflict` (409).

The 33+ `http.StatusBadRequest` (400) returns in handler files are exclusively
**request-parse** failures (malformed JSON body, missing required fields at the
HTTP layer), not error-class mapping from repository errors.

So the issue's central claim — "no error-class discrimination of repository
errors at the HTTP boundary" — stands fully validated.

## Malformed UUID handling — additional gap not in issue body

`DELETE /datastreams/not-a-uuid` returns **204 No Content** (live-tested).

GORM passes the literal string to PostgreSQL which would normally fail with
SQLSTATE `22P02 invalid_text_representation` against a `uuid` column — yet
the response is 204. This implies either (a) GORM/pgx is converting to a
zero-row no-op silently, or (b) the result rowcount path ignores cast errors.
Either way: the client receives the same 204 as for "deleted successfully" and
"resource never existed" and "input was malformed". This is a third
indistinguishable case the issue body doesn't enumerate.

## POST handler claim ("23502/23505/22P02 collapse to 500") — out of scope spot-check

Not in the primary scope of #17 (which targets DELETE), but the issue body
mentions it. Spot-check: `git grep -n "Status.*Created\|StatusInternalServerError" -- internal/api/datastream_handler.go`
shows `CreateDatastream` returns 500 on any `repo.Create` error with
`{"error":"Failed to create datastream"}`. No `errors.As` for `pgconn.PgError`.
Same generic pattern. Claim holds for POST as well, though the recommended fix
is the same `classifyRepoError` helper.

## PUT/PATCH handler claim — confirmed pattern

`git grep -n "Update\|Patch" -- internal/api/router.go` and spot-checking
several `Update<X>` handlers shows the same `if err := h.repo.Update(...); err != nil { render.Status(r, 500); return }` pattern with no error-class
discrimination. Claim holds.

## Conclusion

Issue #17's **core claim** — that all 12 DELETE handlers (and PUT/PATCH/POST
handlers symmetrically) lack error-class discrimination on repository errors,
collapsing FK conflicts and other recoverable error states into opaque 500s —
is fully correct.

Issue #17's **framing** of the status quo overstates the problem in one
direction (missing 404 path is only true for 7 of 12 handlers; the other 5
already do pre-flight 404) and understates it in another (missing IDs in the
7 no-preflight handlers actually return 204, not 500 — silent success rather
than loud error; this is arguably worse for clients implementing
"delete-or-create" idempotency because the 204 "succeeded" leaks no
information about whether the resource was deleted *now* or never existed).

The recommended `classifyRepoError` helper resolves both the validated central
problem and the framing nuances if combined with a uniform pre-flight presence
check (or a switch to `db.Delete(...).RowsAffected == 0 → 404`). The issue's
P2 severity assessment is accurate.
