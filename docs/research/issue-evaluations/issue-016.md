# Issue #16 — Evaluation

**Title:** [bug] DELETE cascade is broken across resource types — m2m join FKs
not cleaned, and natural parent→child relations have no FK at all
**Issue HEAD ref:** `4b99421` · **Eval HEAD ref:** `edc8459` · code drift on
referenced paths: **none** (`git log 4b99421..HEAD -- <paths> ` returns empty)
**Verdict:** **KEEP — VALIDATED.** All three bugs reproduce empirically and/or
statically, with claimed line numbers, SQL constraint names, and code patterns
matching exactly.
**Severity assessed:** **P1 confirmed.** Bug 1 alone justifies P1; Bugs 2 and 3
expand the blast radius.

## Per-bug verdict

### Bug 1 — `?cascade=true` non-functional · **VALIDATED, P1**

Empirical, fresh on HEAD `edc8459`:

```
DELETE /datastreams/$dsId?cascade=true   →  500   "Failed to delete datastream"
DELETE /systems/$sysId?cascade=true      →  500   "Failed to delete system"
DELETE /datastreams/$dsId                →  500   (non-cascade also 500s when m2m row exists)
```

Static: `internal/repository/datastream_repository.go:134-146` and
`control_stream_repository.go:134-145` have **identical** structural defect —
cascade transaction deletes natural-relation children (Observations / Commands)
but never executes `DELETE FROM system_datastreams` /
`DELETE FROM system_controlstreams` before the resource delete. The single
existing FK on the join (`fk_system_datastreams_datastream` /
`fk_system_controlstreams_control_stream`) trips on the resource delete.

`SystemRepository.deleteCascade`
(`internal/repository/system_repository.go:189-235`) exhibits an asymmetric
form of the same defect: it explicitly cleans `system_deployments` (line 226)
and `system_procedures` (line 229) before deleting the system, but **never**
cleans `system_datastreams` / `system_controlstreams` (verified by full-file
grep — both names appear only in read-path `JOIN` clauses at lines 443/452).

### Bug 2 — `DeploymentRepository.Delete` lacks cascade · **VALIDATED, P2**

`internal/repository/deployment_repository.go:111`:
`func (r *DeploymentRepository) Delete(id string) error` — no cascade param.
`internal/api/deployment_handler.go:118-129` — no `?cascade=` parsing. Among
12 repository `Delete` methods, only 3 (`System`, `Datastream`, `ControlStream`)
take a `cascade bool`. Issue's claim of "8 other DELETE handlers without cascade
support: feature, system_event, history, command, observation, procedure,
sampling_feature, property" is exact (Deployment is counted separately as
Bug 2). Total breakdown: 3 with cascade, 9 without.

Severity is P2, not P1, because the missing cascade only manifests when the
deployment has been linked into a system via `POST /deployments/{id}/systems`
or has a sub-deployment via the closure trigger. Unlike Bug 1, this is not
the *only* path to delete a deployment (a deployment with no links can still
be deleted without 500ing).

### Bug 3 — Silent orphaning from missing natural-side FKs · **VALIDATED, latent P2 / surfaces as P1 once Bug 1 is fixed**

Static evidence (full `git grep "foreignKey:" -- internal/model/domains/`):
**only two** active relationship tags exist —
`System.SystemKind` (`SystemKindID → procedures.id`) and
`System.SamplingFeatures` (`ParentSystemID → sampling_features.parent_system_id`).
Every other natural 1:N relation on `Observation`, `Command`, `SystemEvent`,
`SystemHistoryRevision`, `Datastream` (system_id projection), and
`Deployment` (parent_deployment_id) lacks a Go-level relationship tag, so
GORM's `AutoMigrate` does not emit a `REFERENCES` clause. The
`fk-constraints-head-2026-04-30.txt` evidence file — independently captured
during #2 evaluation, reproducible against the live DB — confirms exactly the
absences claimed.

Additionally, `SystemRepository.deleteCascade` does not delete `SystemEvent`
rows (full-file grep returns zero matches for `SystemEvent` /
`system_events`), so once Bug 1 is fixed the system cascade would silently
orphan `system_events` rows unless this is added.

Currently masked because:
- The only POST route for datastreams is `POST /systems/{id}/datastreams`
  (`internal/api/router.go:118-126` — no top-level `POST /datastreams`), so
  every datastream has a `system_datastreams` row, so Bug 1's FK trip fires
  before the natural-side orphaning can manifest.
- The non-cascade DELETE path of `Datastream` *does* succeed if and only if
  the m2m row has been removed by some other path. No such API path exists
  today, so the orphaning channel has no live trigger.

The "worst possible combination" framing in the issue body — "no FK + no
cascade-cleanup on the natural side" — is correct. Bug 1's mask is fragile:
fixing Bug 1 without simultaneously addressing Bug 3 would *introduce* live
orphaning, not resolve it.

## Severity overall · P1 confirmed

Bug 1 alone is P1: the canonical CS workflow `POST system → POST datastream
(under system) → POST observations → DELETE datastream/system?cascade=true` is
unrecoverable through the API. Operators have no way to tear down a test
system other than direct SQL.

Bug 2 expands this to deployments. Bug 3 makes the eventual fix non-trivial:
a Bug-1-only patch that adds m2m cleanup would *unmask* Bug 3 unless natural
children are handled at the same time.

## Recommended remediation review

The issue body proposes three coordinated changes; reviewing each:

| Proposal | Assessment |
|---|---|
| Mirror `SystemRepository`'s `system_deployments` / `system_procedures` cleanup pattern in `DatastreamRepository.Delete` (cascade=true), `ControlStreamRepository.Delete` (cascade=true), and `SystemRepository.deleteCascade` (add `system_datastreams` + `system_controlstreams`) | Correct and minimal; matches existing convention exactly. |
| Add `cascade bool` to `DeploymentRepository.Delete` + handler-side `?cascade=` parsing | Correct API parity with System/Datastream/ControlStream; required for full deployment teardown. |
| Add `mapDeleteError` helper mapping `pgconn.PgError{Code:"23503"}` → 409 Conflict, `gorm.ErrRecordNotFound` → 404, default → 500 | Correct status-code semantics; sanitisation property (no SQL identifier leak) is preserved if the response body uses fixed strings rather than `pgErr.Message`. |
| Approach 3a: Add `foreignKey:`/`references:`/`constraint:OnDelete:RESTRICT` tags to surface natural FKs | Correct schema-side fix; AutoMigrate idempotency means existing tables get the new constraint added on next migration. **Caveat**: if any orphaned rows already exist from prior buggy DELETEs (Bug 1 hasn't allowed any to occur via the API surface, but operator-side direct SQL may have), AutoMigrate's constraint-add will fail. A pre-migration `DELETE FROM observations WHERE datastream_id NOT IN (SELECT id FROM datastreams)` cleanup pass would be prudent. |
| Approach 3b: Have non-cascade `Delete` paths refuse with 409 if children exist | Acceptable fallback if 3a is too invasive; reduces silent-orphan risk without schema migration. |

The recommended path is **3a + the cleanup pass** — schema-correct,
self-documenting, lets DB enforce integrity. 3b is an acceptable mitigation
if the team wants zero schema migration in this PR.

## Adjacent observations (do not require new issues — already covered or out of scope)

1. The current 500 response body `{"error":"Failed to delete X"}` does not
   leak SQL identifiers. This is good. The proposed `mapDeleteError` helper
   should preserve that property — return fixed strings, not `pgErr.Message`
   nor `pgErr.ConstraintName`.
2. `removeSystemFromDeployments`
   (`internal/repository/system_repository.go:267+`) iterates and updates
   `Deployment.SystemIds` JSONB on system delete. This logic is independent
   of the m2m join cleanup and unaffected by this fix.
3. The closure-table triggers for `deployments.parent_deployment_id` and
   `systems.parent_system_id` are correctly out of scope per the issue body's
   own scope statement; the closure logic is in
   `internal/repository/closure.go` and is functioning correctly per #2's
   evaluation.

## Recommendation

**KEEP. Validated as a high-quality umbrella issue.** All three sub-bugs are
real, line numbers are accurate, severity (P1) is justified, and the proposed
remediation is structurally correct. The cross-references to #2 (origin) and
#15 (m2m structural concern) are appropriate. No edits or scope changes
required to the issue body.

## Evidence

- [`docs/research/evidence/issue-016/static-analysis-2026-04-30.md`](../evidence/issue-016/static-analysis-2026-04-30.md)
- [`docs/research/evidence/issue-016/live-test-2026-04-30.md`](../evidence/issue-016/live-test-2026-04-30.md)
- [`docs/research/evidence/issue-016/spec-authority-2026-04-30.md`](../evidence/issue-016/spec-authority-2026-04-30.md)
- Reuses: [`docs/research/evidence/issue-002/fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt)
- Reuses: [`docs/research/evidence/issue-002/delete-tests-head-2026-04-30.txt`](../evidence/issue-002/delete-tests-head-2026-04-30.txt)
