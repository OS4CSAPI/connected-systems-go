# Report 02 — Datastream `applyFilters` dangling `unique_identifier` SQL

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-02-datastream-dangling-unique-identifier-sql.md`](../upstream-issues/plan-02-datastream-dangling-unique-identifier-sql.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#4** (Datastream `applyFilters` dangling `unique_identifier` SQL) |
| Source fork issue | `OS4CSAPI/connected-systems-go#12` (closed; this finding is **Adjacent Finding A** of the eval, not the body of #12) |
| Research plan | [`../upstream-issues/plan-02-datastream-dangling-unique-identifier-sql.md`](../upstream-issues/plan-02-datastream-dangling-unique-identifier-sql.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P2** — documented filter parameter unconditionally broken on fresh deploys (corrected from backlog's "P3-latent") |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Defect persists on both static and live.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git show upstream/main:internal/repository/datastream_repository.go |
    Select-String -Pattern 'unique_identifier|applyFilters' -Context 3,3

  query := r.db.Model(&domains.Datastream{})
> query = r.applyFilters(query, params, systemID)

> func (r *DatastreamRepository) applyFilters(query *gorm.DB,
      params *queryparams.DatastreamsQueryParams, systemID *string) *gorm.DB {
      if len(params.IDs) > 0 {
>         query = query.Where("id IN ? OR unique_identifier IN ?",
              params.IDs, params.IDs)
      }
```

```text
$ git show upstream/main:internal/model/domains/datastream.go |
    Select-String -Pattern 'CommonSSN|UniqueIdentifier|unique_identifier'
(no matches — column genuinely gone from the model)

$ git log --oneline upstream/main -- internal/model/domains/datastream.go |
    Select -First 5
2dc09f7 oh gorm/go... adding `json:-`
d2d1347 Cleaned up data/controlstream's SystemLink
1562201 update datastreams           <-- removed CommonSSN embedding
dacae7b Bug fixes and clean up
f2cf1c3 Adding the other resources along with e2e tests
```

```text
$ foreach ($u in @('…/datastreams?id=foo','…/datastreams?id=urn:test:ds:nope')) {
    curl.exe -s -o NUL -w '%{http_code}' --max-time 10 $u
  }
.../datastreams?id=foo                    -> HTTP 500
.../datastreams?id=urn:test:ds:nope       -> HTTP 500

$ curl.exe -s '…/datastreams?id=foo'
{"error":"Internal server error"}
```

**Conclusion:** As of `df6da0d`, `applyFilters` still references the
deleted column. Two `?id=` probes against the live cs-go-head deployment
return HTTP 500 deterministically. Defect is live.

> **Live-evidence caveat.** cs-go-head is the OS4CSAPI fork's deployment.
> The relevant lines of `datastream_repository.go` are byte-for-byte
> identical between `upstream/main` (`df6da0d`) and the fork — so
> reproduction equivalence holds. A pure-upstream deployment is being
> considered for future reports; for this report static + live-on-fork
> is sufficient because the affected code is identical.

## 2. Static evidence

Source: [`../evidence/issue-012/static-analysis-2026-04-30.md`](../evidence/issue-012/static-analysis-2026-04-30.md) §3 (refreshed in §1 above).

`internal/repository/datastream_repository.go` — `applyFilters`:

```go
func (r *DatastreamRepository) applyFilters(query *gorm.DB,
    params *queryparams.DatastreamsQueryParams, systemID *string) *gorm.DB {

    if len(params.IDs) > 0 {
        query = query.Where("id IN ? OR unique_identifier IN ?",
            params.IDs, params.IDs)
    }
    // …
}
```

`internal/model/domains/datastream.go` — post-`1562201`, `Datastream`
embeds only `Base` (no `CommonSSN`, no `UniqueIdentifier` field, no
`gorm:"...uniqueIndex"` tag for a `unique_identifier` column).

The struct field that backed `unique_identifier` was removed in
**`1562201` ("update datastreams")**, but the SQL `WHERE` clause was
not updated. `git grep` for `CommonSSN|UniqueIdentifier|unique_identifier`
in `datastream.go` returns zero hits. The column is genuinely gone from
the model.

## 3. Live evidence

Source: [`../evidence/issue-012/live-test-2026-04-30.md`](../evidence/issue-012/live-test-2026-04-30.md) (refreshed in §1 above).

```text
GET /datastreams?id=foo                       -> HTTP 500
GET /datastreams?id=urn:test:ds:nope          -> HTTP 500
GET /datastreams?id=4300e090-7893-4c01-...    -> HTTP 500   (real id, in DB)
Body: {"error":"Internal server error"}
```

All three return 500 unconditionally on the live cs-go-head deployment.

**Caveat — additive `AutoMigrate` semantics.** GORM's `AutoMigrate` does
**not** drop columns when a struct field is removed. A long-running
deployment that started **before** `1562201` (`f2cf1c3..1562201^` range)
and was never wiped may still carry the legacy `unique_identifier`
column, in which case the `?id=` filter would silently pass through to
the (nonsense) `OR` clause without erroring. The defect is still real
on those deployments — the SQL reference is wrong relative to the
declared model — but the empirical 500 only reproduces on **fresh
deploys** (or any deploy where the column has been dropped). Worth
mentioning so the maintainer doesn't dismiss the reproducer because
their dev DB happens to retain the column.

## 4. Spec authority

This finding is **not primarily a spec-conformance issue** — it is a
self-inconsistency between cs-go's domain model and its own SQL.
Spec authority plays a supporting role only.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 Datastream schema (`baseStream.json`, `dataStream.json` from the Part 2 OpenAPI bundle) | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Supporting context: confirms the canonical spec defines no `uid` property on Datastream; dropping the column entirely (commit `1562201`'s direction) was the spec-faithful choice. Justifies the recommended fix shape (delete the OR-clause, do not re-add a column). |

**Out-of-scope sources (not cited in the public extract):**
- OAS 3.0.3 — irrelevant; not an OpenAPI conformance issue.
- OGC 19-072 — irrelevant; server-internal SQL bug, not public-API contract.
- OGC 23-001 — irrelevant; affects only Datastream.

## 5. Alternatives considered (internal-only)

Two viable fix shapes:

### Option A — Remove the `OR unique_identifier IN ?` clause (recommended)

```go
if len(params.IDs) > 0 {
    query = query.Where("id IN ?", params.IDs)
}
```

- **Authoring delta:** one-line change.
- **Risk:** zero. The dropped clause references a column that does not
  exist in the post-`1562201` schema; removing it cannot affect any
  query that currently succeeds.
- **Spec fidelity:** matches `1562201`'s direction (Datastream identified
  by `id` only; spec-faithful per OGC 23-002).

### Option B — Remove the entire `if len(params.IDs) > 0 { … }` block

If `id`-based filtering belongs in a different layer (handler-side
short-circuit to `GetByID`, or a separate filter dispatcher), the
whole block could be dropped.

- **Authoring delta:** small, but architectural.
- **Risk:** changes the handler/repository boundary. Not our call to
  make.

**Ranking: Option A.** Lead with A only; B is architectural and beyond
the scope of "fix the dangling reference."

## 6. Recommended fix

**Lead with Option A** — remove the `OR unique_identifier IN ?` clause:

```go
if len(params.IDs) > 0 {
    query = query.Where("id IN ?", params.IDs)
}
```

Implementation surface: `internal/repository/datastream_repository.go`,
inside `applyFilters` (single line).

## 7. Scope guard

What NOT to touch as part of this filing:

- The `Datastream` domain model (correct as-is post-`1562201`).
- `AutoMigrate` (no schema migration needed; column is already gone
  from the model).
- Any other repository's filter logic.
- The `ControlStream` repository — `unique_identifier` may legitimately
  still exist there (see backlog item #5 / report-03 for the related
  ControlStream sibling concern).
- Re-introduction of any `uid` field on Datastream — explicitly out
  of scope; the spec direction is no-`uid`-on-Datastream.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#12` was filed against an empirical
  observation that almost certainly reproduced on a pre-`1562201`
  build. The fork-side closure rationale: HEAD has moved past the
  premise (`uid` field gone entirely; UID-uniqueness question moot).
- This finding is **Adjacent Finding A** logged in the #12 evaluation,
  not the body of #12. The other adjacent findings (B: DELETE 500; C:
  three orphaned test datastreams in the live DB) are tracked
  separately.
- Live test seeded three datastreams that remain in the cs-go-head DB
  due to the unrelated DELETE 500 defect. Mentioning here for fork
  bookkeeping; out of scope for this filing.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Frame as defect or refactoring miss? | **Cleanup-miss framing**, with HTTP-500 impact front-loaded so severity is unambiguous. |
| Severity P2 or P3? | **P2.** Plan's backlog-accuracy flag confirmed by re-verification — fresh-deploy `?id=` queries return 500 deterministically. Step 4.4 must update the backlog entry. |
| Mention additive-`AutoMigrate` complication? | **Yes.** Single-line caveat under Live evidence; pre-empts dismissal-by-stale-dev-DB. |
| Lead with Option A or A+B? | **A only.** B is architectural; not our call. |
| Cite eval/evidence paths? | **Yes**, as "validation chain" footer links. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P2] /datastreams ?id= filter returns HTTP 500 on fresh deploys: applyFilters references dropped 'unique_identifier' column`

**Labels:** `bug`

---

### Context

Commit **`1562201` ("update datastreams")** removed the `CommonSSN`
embedding from `Datastream`, and with it the `unique_identifier`
column (this was the spec-faithful direction — OGC 23-002 / CSAPI
Part 2 defines no `uid` on Datastream). One SQL site in
`internal/repository/datastream_repository.go` was missed during that
cleanup: `applyFilters` still issues
`Where("id IN ? OR unique_identifier IN ?", ...)` against a column
that no longer exists in the post-`1562201` schema.

The result: any deployment built fresh from current `upstream/main`
returns HTTP 500 on the documented `?id=` filter parameter on
`/datastreams`.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

`?id=` on `/datastreams` returns HTTP 500 on any deployment built
fresh from current `upstream/main` because the WHERE clause references
a column (`unique_identifier`) that no longer exists in the schema.
The SQL is referentially incorrect relative to the model regardless of
whether a particular long-running deployment happens to retain the
column from a pre-`1562201` `AutoMigrate`.

### Static evidence

`internal/repository/datastream_repository.go`:

```go
func (r *DatastreamRepository) applyFilters(query *gorm.DB,
    params *queryparams.DatastreamsQueryParams, systemID *string) *gorm.DB {

    if len(params.IDs) > 0 {
        query = query.Where("id IN ? OR unique_identifier IN ?",
            params.IDs, params.IDs)
    }
    // …
}
```

`internal/model/domains/datastream.go` (post-`1562201`): `Datastream`
embeds only `Base`. There is no `UniqueIdentifier` / `unique_identifier`
field, no `CommonSSN` embedding, no `gorm:"...uniqueIndex"` tag for
that column.

```text
$ git grep -nE 'CommonSSN|UniqueIdentifier|unique_identifier' \
    upstream/main -- internal/model/domains/datastream.go
(no matches)

$ git log --oneline upstream/main \
    -- internal/model/domains/datastream.go | head -5
2dc09f7 oh gorm/go... adding `json:-`
d2d1347 Cleaned up data/controlstream's SystemLink
1562201 update datastreams           <-- column removed here
dacae7b Bug fixes and clean up
f2cf1c3 Adding the other resources along with e2e tests
```

### Live evidence

```text
$ curl -s -o /dev/null -w '%{http_code}\n' '<deployment>/datastreams?id=foo'
500

$ curl -s -o /dev/null -w '%{http_code}\n' \
    '<deployment>/datastreams?id=urn:test:ds:nope'
500

$ curl -s '<deployment>/datastreams?id=foo'
{"error":"Internal server error"}
```

**Caveat:** GORM `AutoMigrate` does not drop columns when a struct
field is removed. Long-running deployments that started before
`1562201` and were never wiped may still carry the legacy
`unique_identifier` column, in which case the `?id=` query would
pass through without erroring. The SQL is still wrong relative to the
declared model in that case; the HTTP 500 reproduces deterministically
only on fresh deploys.

### Spec authority

The canonical CSAPI Part 2 Datastream schema (OGC 23-002 —
`baseStream.json` / `dataStream.json` in the Part 2 OpenAPI bundle)
defines **no `uid` property on Datastream** (UID is a System-level
concept; Datastreams are identified solely by their server-side `id`).
This supports the conclusion that `1562201`'s direction (drop the
column entirely) was correct, and the fix here is to remove the
dangling SQL clause — not re-introduce a column.

### Recommended fix

In `internal/repository/datastream_repository.go`, replace:

```go
query = query.Where("id IN ? OR unique_identifier IN ?",
    params.IDs, params.IDs)
```

with:

```go
query = query.Where("id IN ?", params.IDs)
```

One-line change. Risk is zero — the dropped clause references a
column that does not exist in the post-`1562201` schema, so removing
it cannot affect any query that currently succeeds.

### Scope guard

Out of scope for this filing:

- The `Datastream` domain model (correct as-is post-`1562201`).
- `AutoMigrate` / schema migrations (column already gone from the model).
- Any other repository's filter logic.
- `ControlStream` — `unique_identifier` may legitimately still exist
  there if `CommonSSN` is still embedded; that is a separate concern.

### Validation chain (links)

Internal evaluation and evidence (public on the OS4CSAPI fork):

- Synthesis report:
  `OS4CSAPI/connected-systems-go:main/docs/research/upstream-issue-reports/report-02-datastream-dangling-unique-identifier-sql.md`
- Rigour-first evaluation:
  `OS4CSAPI/connected-systems-go:main/docs/research/issue-evaluations/issue-012.md`
  (this finding is logged as **Adjacent Finding A**)
- Static analysis:
  `OS4CSAPI/connected-systems-go:main/docs/research/evidence/issue-012/static-analysis-2026-04-30.md`
- Live test:
  `OS4CSAPI/connected-systems-go:main/docs/research/evidence/issue-012/live-test-2026-04-30.md`

Cross-reference: original fork-side issue
`OS4CSAPI/connected-systems-go#12` (closed; the body of #12 was about
Datastream UID uniqueness, not this SQL — see the eval for context).

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
