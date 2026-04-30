# Issue #12 — Evaluation

**Issue:** [#12](https://github.com/OS4CSAPI/connected-systems-go/issues/12) — `Datastream unique_identifier enforces global uniqueness — should be scoped per parent system`
**Reporter framing:** P2-Important, category API Design, ownership upstream. Evidence cited as live integration testing during OSHConnect-Python publisher fleet migration.
**Repo @:** `c9e4fcf` (`origin/main`).
**Endpoint:** `https://129-80-248-53.sslip.io/csapi-go-head` (Date `Thu, 30 Apr 2026`).

## Verdict

**Not a defect on cs-go HEAD.** The reported behaviour does not reproduce: cs-go HEAD enforces no UNIQUE constraint on Datastream UIDs, in any scope. Two different parent systems can host datastreams with identical `uid`, identical `name`, and identical body — both POSTs return HTTP 201 with distinct server-assigned ids. The specific code change the reporter probably observed against an earlier revision was already applied in commit `1562201` ("update datastreams"), which removed the `CommonSSN` embedding from `Datastream` and with it the `unique_identifier` column.

The maintainer cleanup in `1562201` went **further** than this issue requests: it dropped the column entirely (issue's Option B) rather than scoping it to `(system_id, unique_identifier)` (issue's Option A). I argue Option B is the spec-faithful choice — the canonical CSAPI Part 2 Datastream JSON schema has no `uid` property. See evidence/spec-authority for the schema citations.

I therefore recommend the issue be closed as already-fixed / not-a-defect-on-HEAD, with a clear acknowledgement of two adjacent findings the static + live work surfaced.

## Evidence

- **Static:** [evidence/issue-012/static-analysis-2026-04-30.md](../evidence/issue-012/static-analysis-2026-04-30.md) — `Datastream` embeds only `Base`, has no `UniqueIdentifier`/`uid` field, and no `gorm:"uniqueIndex"` tag anywhere. Git history shows `CommonSSN` was removed from the embedding in commit `1562201`. Adjacent finding: `datastream_repository.go:164` still references `unique_identifier` in SQL — dangling after the cleanup.
- **Live:** [evidence/issue-012/live-test-2026-04-30.md](../evidence/issue-012/live-test-2026-04-30.md) — three POSTs of the issue's exact reproducer body, two under different parent systems and one re-POST under the same parent system, all returned HTTP 201 with distinct server-side ids. The dangling SQL reference at line 164 was probed (`?id=foo` on `/datastreams`) and confirmed to produce HTTP 500. DELETE on the seeded datastreams also returned HTTP 500 (separate, unrelated bug).
- **Spec:** [evidence/issue-012/spec-authority-2026-04-30.md](../evidence/issue-012/spec-authority-2026-04-30.md) — the canonical `baseStream.json` and `dataStream.json` schemas at `opengeospatial/ogcapi-connected-systems@master` define no `uid` property on Datastream. Datastreams are identified solely by `id`. `uid` is a System-level concept.

## Where the issue's framing came from

The reporter most likely tested the OSHConnect-Python publisher against a build of cs-go between commits `f2cf1c3` ("Adding the other resources along with e2e tests") and `1562201` ("update datastreams"). In that range, `Datastream` embedded `CommonSSN`, which carried `UniqueIdentifier UniqueID gorm:"type:varchar(255);uniqueIndex" json:"uid"`. GORM's `uniqueIndex` tag is single-column and table-scoped, so two datastreams with the same UID anywhere in the table — regardless of parent system — would have produced the constraint violation the reporter described. The reproducer steps and the predicted HTTP 500 are both consistent with that pre-`1562201` schema.

So the reporter's empirical observation was almost certainly correct at the time of testing. By the time the issue was filed, `1562201` had landed and the column was gone. This is a timing/state mismatch, not a reporting error on the publisher's part.

## Adjacent findings (out of scope — recommend separate issues)

These were uncovered while evaluating #12 but are not the defect #12 describes.

### A. Dangling `unique_identifier` SQL reference at `datastream_repository.go:164`

```go
if len(params.IDs) > 0 {
    query = query.Where("id IN ? OR unique_identifier IN ?", params.IDs, params.IDs)
}
```

The struct field that backed `unique_identifier` was removed in `1562201`, but this SQL `WHERE` clause was not updated. On any deployment built from `c9e4fcf` where the column does not exist, every `?id=` query against `/datastreams` returns HTTP 500. Live confirmation:

```
GET /datastreams?id=foo                       → 500
GET /datastreams?id=urn:test:ds:something     → 500
GET /datastreams?id=4300e090-...               → 500   (real id, in DB)
```

Severity: P2 (a documented filter parameter is unconditionally broken). Fix is small: remove the `OR unique_identifier IN ?` clause and pass `params.IDs` once.

This may interact with the existing `id`-filter discussion in #1 / #7. Worth filing as its own report unless one of those already covers it.

### B. DELETE `/datastreams/{id}` returns HTTP 500

Confirmed during cleanup of the live-test seeds. Three attempts (with and without `?cascade=true`) all returned `{"error":"Failed to delete datastream"}` HTTP 500. Distinct from finding A — the failing path is `repo.Delete`, not `applyFilters`. Out of scope here; recommend filing as a separate issue with a focused reproducer.

A consequence of B: the three datastreams seeded for this evaluation (`4300e090-...`, `95146016-...`, `26a80a70-...`) remain in the HEAD database and cannot be removed via the public API. They will need a maintainer-side wipe.

## Refinement to issue framing

The issue's framing depends on a property (`Datastream.uid`) that the canonical schema does not define. Once that is registered, the global-vs-per-system question is moot: there is nothing for the schema to scope. cs-go HEAD's current state — Datastream identified by `id` only, no UID round-trip — is in fact spec-faithful, and it already implements the issue's Option B.

The issue's acceptance criteria should also be reviewed: criterion (1c) ("two datastreams under same system with same UID still produce constraint violation") is not met on HEAD either, but I do not believe this is a defect because the spec does not require any UID-level uniqueness on Datastreams.

## Severity

Recommend: **closed as not-a-defect / already-addressed-by-1562201**. The reporter's empirical evidence was almost certainly accurate against an earlier revision; HEAD has moved past it. The two adjacent findings (A and B above) deserve their own issues.

## Recommendations to issue thread

- Acknowledge the reporter's empirical observation was likely correct against a pre-`1562201` build.
- State that on `c9e4fcf` the `unique_identifier` column has been removed entirely, so the constraint violation no longer reproduces.
- Note that the canonical Datastream JSON schema has no `uid` property; this places HEAD's state firmly within the spec-faithful range.
- Flag finding A (dangling `unique_identifier` SQL reference at `datastream_repository.go:164`) as a separate, currently-active P2 to file.
- Flag finding B (`DELETE /datastreams/{id}` → 500) as a separate issue.
- Note that three test datastreams remain in the HEAD database due to finding B and ask whether a maintainer-side wipe is acceptable.

## Methodology notes

1. **Git history check is high-leverage when the static state contradicts the live report.** When the model has none of the fields the issue describes, but the reporter's reproducer is too specific to be a fabrication, the most likely explanation is a state mismatch. Two `git log` commands answered this in seconds.
2. **GORM `AutoMigrate` is additive.** Removing a struct field does not drop the column. Issue #12's reporter may have been testing against a long-running deployment where the legacy `unique_identifier` column persisted past the model change — the constraint would still fire there even though new fresh installs don't have the column. The HEAD endpoint as of 2026-04-30 evidently is a fresh install (no constraint, plus the dangling-reference HTTP 500), so we cannot directly observe that hybrid state, but it's worth flagging in our internal notes that "fresh install vs upgraded install" can produce divergent behaviour for deployer reports.
3. **Spec-property vacuum is its own answer.** When neither side of an "X should be A vs B" debate corresponds to a property the spec defines (`uid` on Datastream), the spec-derived answer is "the question is malformed". This is closely related to the SensorHub-as-baseline pattern: a comparator implementation introduces a property; the issue takes the property as given; the spec says nothing about it; the spec-compliance call collapses.
4. **Adjacent findings during a refute-the-claim evaluation.** Refuting the headline claim doesn't end the evaluation. The probes that establish the refutation can surface other defects (here, finding A from a one-line SQL inspection and finding B from cleanup). Recording them as separate items keeps each issue thread focused while preserving the investigative work.
