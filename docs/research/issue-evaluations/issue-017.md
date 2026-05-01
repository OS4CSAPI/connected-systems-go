# Issue #17 — Evaluation

**Title:** [enhancement] All 12 DELETE handlers map every repository error to
HTTP 500 — should distinguish 404 (not found), 409 (FK conflict / unique
violation), 400 (validation), 500 (true internal)
**Issue HEAD ref:** session-current · **Eval HEAD ref:** `0255a10`
**Verdict:** **KEEP — VALIDATED with refinements.** Core thesis fully correct;
status-quo framing in the issue body is partially inaccurate and should be
amended in a follow-up edit (or the comment can stand as the public correction).
**Severity assessed:** **P2 confirmed.**

## Validity per claim

| Issue-body claim | Verdict | Evidence |
|---|---|---|
| 12 `r.Delete("/", ...)` routes in `internal/api/router.go` | **TRUE** | `router.go:96,108,132,138,155,172,189,200,212,228,239,251` |
| All 12 wired to `<X>Handler.Delete<X>` | **TRUE** | 12 handler functions enumerated; perfect 1:1 |
| "no `errors.As(*pgconn.PgError)`, no SQLSTATE inspection" | **TRUE** | `git grep` returns zero matches for `errors.As`/`pgconn`/`SQLSTATE` in `internal/api/` |
| "no 404 path for missing resources" | **PARTIALLY FALSE** | 5 of 12 handlers (Command, Feature, Observation, SystemEvent, SystemHistory) implement pre-flight `GetByID` 404; verified at lines 182-186 / 199-205 / 155-159 / 238-241 / 146-149 |
| "no 409 for FK conflicts" | **TRUE** | Live: `DELETE /datastreams/<fk-conflicting-id>` → 500; zero `StatusConflict` in handlers |
| "everything → 500 indistinguishably" | **MORE NUANCED** | Live: missing ID → **204** (not 500) for the 7 no-preflight handlers because GORM returns nil for 0-row deletes; FK conflict → 500 (correct in scope); malformed UUID → **204** (additional gap not in issue body) |
| Sanitised body (no SQL leak) — refutes #2's claim | **TRUE** | Verified: response body is fixed `{"error":"Failed to delete X"}` with no `pgErr.Message` interpolation |
| PUT/PATCH handlers same pattern | **TRUE** (spot-check) | `Update<X>` handlers spot-checked across `datastream_handler`, `system_handler`, `deployment_handler`; all `if err != nil → 500` |
| POST handlers collapse 23502/23505/22P02 to 500 | **TRUE** (spot-check) | `CreateDatastream` returns 500 generic for all `repo.Create` errors |
| GET-by-id "already handle ErrRecordNotFound → 404 in most places" | **TRUE** | Spot-checked; consistent with the 5 preflight DELETE handlers re-using existing 404 paths |

## Refinements to status quo (not in issue body)

1. **Inconsistent 404-vs-204 split for missing resources.**
   - 5 of 12 (Command, Feature, Observation, SystemEvent, SystemHistory) →
     **404** via pre-flight `GetByID`
   - 7 of 12 (ControlStream, Datastream, Deployment, Procedure, Property,
     SamplingFeature, System) → **204** because GORM `db.Delete` returns nil
     when 0 rows match
   - Both are RFC 9110-compliant in isolation (§9.3.5 idempotency permits
     204 for repeated DELETE on absent resource), but the *inconsistency
     within one API surface* is the real defect.
2. **Malformed UUIDs return 204, not 400.** A non-UUID path parameter
   (`/datastreams/not-a-uuid`) should produce a `22P02 invalid_text_representation`
   from PostgreSQL when compared against a `uuid` column, but the response is
   204. Either GORM/pgx is silently no-op'ing or the cast error is being
   swallowed — either way clients can't distinguish "deleted", "never
   existed", or "structurally invalid input".
3. **`SystemEvent` cleanup was missed in #16's analysis.** Cross-referencing:
   `SystemEvent` cleanup was correctly noted as missing from
   `SystemRepository.deleteCascade`, but `DeleteEventByID` itself is one of
   the 5 handlers that already does the right pre-flight 404. Not a defect
   in #17's scope; just confirms #16's adjacent finding.

## Severity

**P2 confirmed.** Not a data-correctness issue. UX/observability/contract
issue. Independent of #16 (which fixes cascade). Together #16 and #17
produce the right end state: cascade works, and when it can't (or fails for
other reasons) the client gets an actionable status code.

## Remediation review

The proposed `classifyRepoError` helper:

```go
func classifyRepoError(err error) (status int, msg string) {
    switch {
    case errors.Is(err, gorm.ErrRecordNotFound): return 404, "Resource not found"
    case pgcode == "23503": return 409, "Resource has dependent rows…"
    case pgcode == "23505": return 409, "Resource conflicts with existing entry…"
    case pgcode == "23502": return 400, "Required field is missing"
    default:                return 500, "Internal error"
    }
}
```

is **structurally correct**. Two refinements worth recording for the fix PR:

1. **`gorm.ErrRecordNotFound` is rarely returned by `db.Delete`.** GORM
   returns nil for 0-row deletes; `ErrRecordNotFound` is a `First` / `Take`
   thing. So the `errors.Is(err, gorm.ErrRecordNotFound)` branch in
   `classifyRepoError` won't fire on the DELETE path unless the handler
   structure changes. The 7 no-preflight handlers should additionally check
   `result.RowsAffected == 0 → 404`, or extend the pre-flight pattern from
   the 5 already-correct handlers. The fix PR should make this explicit:
   `classifyRepoError` is necessary but not sufficient for unifying the
   404 channel.

2. **Malformed-UUID → 400.** Either add `_, err := uuid.Parse(id)` at the
   handler entry (returning 400 for parse failure), or rely on PG `22P02`
   surfacing as 500-default and adding a `case "22P02": return 400` arm.
   The handler-side parse is cleaner because it short-circuits before any DB
   round trip.

3. **PG SQLSTATE `23514` (check constraint).** Worth adding a 400 arm in
   addition to 23502, since cs-go uses some `CHECK` constraints
   (verifiable via `\d+` on tables, but out of scope to enumerate here).
   Optional; default-500 is acceptable fallback.

## Recommendation

**KEEP. Validated as a high-quality enhancement issue.** The central
recommendation (introduce `classifyRepoError` and apply uniformly) is exactly
right. The status-quo framing in the issue body should be amended to reflect
the 5/7 preflight split and the 204/500/204 (missing/conflict/malformed)
behavior actually observed — but this is a body-clarity nit, not a defect in
the proposal. The validation comment on the issue documents these
refinements publicly.

## Evidence

- [`docs/research/evidence/issue-017/static-analysis-2026-04-30.md`](../evidence/issue-017/static-analysis-2026-04-30.md)
- [`docs/research/evidence/issue-017/live-test-2026-04-30.md`](../evidence/issue-017/live-test-2026-04-30.md)
- [`docs/research/evidence/issue-017/spec-authority-2026-04-30.md`](../evidence/issue-017/spec-authority-2026-04-30.md)
