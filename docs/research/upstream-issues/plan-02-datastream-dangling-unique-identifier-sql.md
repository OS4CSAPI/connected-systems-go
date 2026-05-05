# Research Plan 02 — Datastream `applyFilters` dangling `unique_identifier` SQL

> **🛑 Mandatory pre-work — read every session before touching this plan or
> its corresponding report:**
>
> 1. Open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm the spec/standard sources cited below match the
>    canonical entries there. **Do not self-source references.** If a
>    citation is needed that is not in that list, stop and surface the
>    gap to the user before proceeding.
> 2. Re-read this entire plan file. Do not work from memory.
> 3. Re-read the source eval and evidence files linked in §3 before
>    drafting the report.

---

## 1. Backlog entry

From [`../upstream-followup-backlog.md`](../upstream-followup-backlog.md)
item **4**, copied verbatim:

> ### 4. Datastream `applyFilters` dangling `unique_identifier` SQL
> - **Source:** Issue #12 closure (finding A).
> - **Category:** Defect (P3 — latent / unreachable but wrong).
> - **Summary:** `internal/repository/datastream_repository.go` `applyFilters`
>   still contains a `unique_identifier` filter clause on a column that was
>   removed from the Datastream domain model. Currently unreachable (no handler
>   wires it), but will silently fail at runtime if any code path adds it back.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **⚠️ Backlog accuracy flag.** The backlog's "P3 — latent / unreachable"
> framing is **incorrect** per the live evidence in
> [`../evidence/issue-012/static-analysis-2026-04-30.md`](../evidence/issue-012/static-analysis-2026-04-30.md)
> §3 and the live test (§3 below). The dangling clause **is reachable**:
> `?id=` on `/datastreams` is a documented filter parameter, the handler
> wires it, and every such query returns HTTP 500 against fresh
> deployments. Severity is **P2** (documented filter parameter
> unconditionally broken on fresh deploys), not P3-latent.
> Step 4.4 must update the backlog entry to match this when the issue is
> filed.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#12` — closed during the closure pass.
  This finding is an **adjacent finding** logged in the #12 evaluation,
  not part of the body of #12 itself. Issue #12 was about
  `Datastream.uid` global uniqueness; this is about a dangling SQL
  reference left over after the cleanup that resolved #12's premise.
- No other fork issues bundle into this filing.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-012.md`](../issue-evaluations/issue-012.md) | Full evaluation. The dangling SQL is documented as **Adjacent Finding A**. Severity assessment: P2. Recommended fix: remove the `OR unique_identifier IN ?` clause. |
| [`../evidence/issue-012/static-analysis-2026-04-30.md`](../evidence/issue-012/static-analysis-2026-04-30.md) | §3 carries the verbatim `internal/repository/datastream_repository.go` excerpt around the offending line, plus the git-history context: commit `1562201` removed the `CommonSSN` embedding (and thus the column) but did not update this WHERE clause. |
| [`../evidence/issue-012/live-test-2026-04-30.md`](../evidence/issue-012/live-test-2026-04-30.md) | `?id=foo`, `?id=urn:test:ds:...`, and `?id=<real-uuid>` all returned HTTP 500 against the live `csapi-go-head` deployment. |
| [`../evidence/issue-012/spec-authority-2026-04-30.md`](../evidence/issue-012/spec-authority-2026-04-30.md) | Background only — confirms the canonical CSAPI Part 2 schemas have no `uid` on Datastream, supporting the conclusion that `1562201`'s direction (drop the column entirely) was correct, leaving the dangling SQL as the only thing that needs fixing. |

## 4. Maintainer triage signal

**None.** This finding was surfaced by our evaluation of #12; the upstream
maintainer (`SomethingCreativeStudios`) has not seen it. Implications:

- Tone: **first-time surfacing**, evidence-led.
- Frame as a **byproduct of the partial cleanup in `1562201`** rather
  than a fresh defect — the maintainer already chose to drop the column
  entirely; we're reporting that one site got missed.

## 5. Upstream commits relevant to the area

- **`1562201`** — *"update datastreams"*. Removed `CommonSSN` embedding
  from `Datastream`, dropping the `unique_identifier` column. This is
  the commit that created the dangling SQL state and should be cited
  in the upstream issue body as the origin commit.
- **`c9e4fcf`** — fork-side eval-time HEAD; the WHERE clause was still
  present here.
- Re-verification (Step 4.2) must capture `upstream/main` HEAD SHA at
  filing time and confirm `datastream_repository.go` still contains the
  clause.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'
git show upstream/main:internal/repository/datastream_repository.go | Select-String -Pattern 'unique_identifier|applyFilters' -Context 3,3
git log --oneline upstream/main -- internal/repository/datastream_repository.go | Select-Object -First 5
git log --oneline upstream/main -- internal/model/domains/datastream.go | Select-Object -First 5
git show upstream/main:internal/model/domains/datastream.go | Select-String -Pattern 'CommonSSN|UniqueIdentifier|unique_identifier'
```

```pwsh
# Live re-verification against fresh-deploy semantics:
curl.exe -s -o $null -w 'HTTP %{http_code}\n' 'https://129-80-248-53.sslip.io/csapi-go-head/datastreams?id=foo'
curl.exe -s -o $null -w 'HTTP %{http_code}\n' 'https://129-80-248-53.sslip.io/csapi-go-head/datastreams?id=urn:test:ds:nope'
```

Expected on unchanged state:
- `git show … datastream_repository.go` still contains:
  `query = query.Where("id IN ? OR unique_identifier IN ?", params.IDs, params.IDs)`
- `git show … datastream.go` returns **zero hits** for `CommonSSN` /
  `UniqueIdentifier` / `unique_identifier` (column genuinely gone).
- Both `curl` probes return **HTTP 500** (column not found).

If the live probes return 200 instead of 500, the deployment may have
inherited the column from a pre-`1562201` `AutoMigrate` (additive
behavior). The defect remains real for fresh deploys and the SQL
reference is still wrong; note in the report but do not reframe.

If the WHERE clause is gone on `upstream/main` HEAD, **stop drafting**
— defect already fixed.

## 7. Spec-authority sources

This finding is **not** primarily a spec-authority issue — it is a
self-inconsistency between the cs-go domain model and its own SQL.
Spec authority plays a supporting role only.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 (Dynamic Data) Datastream schema (`baseStream.json`, `dataStream.json` from the Part 2 OpenAPI bundle) | "OGC API - Connected Systems - Part 2: Dynamic Data" and "OGC API - Connected Systems - Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Supporting context only — confirms the canonical spec has no `uid` on Datastream, so dropping the column (rather than re-introducing it) is the spec-faithful direction. Justifies the recommended fix shape (delete the OR-clause, do not re-add a column). |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — irrelevant; this is not an OpenAPI schema-conformance issue.
- OGC 19-072 (OGC API – Common) — irrelevant; this is a server-side
  internal SQL bug, not a public-API contract issue.
- OGC 23-001 (Part 1) — irrelevant; this affects only Datastream.

## 8. Open questions

1. **Frame as defect or refactoring miss?** The maintainer explicitly
   chose Option B (drop the column) when applying `1562201`; framing as
   "you missed one site during the cleanup" is more accurate than
   "you have a bug." **Tentative answer: frame as cleanup-miss, with
   the impact (HTTP 500 on a documented filter for fresh deploys)
   front-loaded so severity is clear.**
2. **Severity declaration.** Backlog says P3-latent; eval and live
   evidence say P2. **Tentative answer: file as P2** with explicit
   correction-of-our-own-prior-misreading note in the upstream issue's
   "Context" section so the maintainer doesn't see contradictory
   framings if they read both this and the original #12.
3. **Mention the additive-`AutoMigrate` complication?** Long-running
   pre-`1562201` deployments may have inherited the column and won't
   reproduce the 500. **Tentative answer: yes, mention as a
   single-line caveat under Live evidence** — otherwise maintainer
   may dismiss the reproducer because their dev DB happens to have
   the column. Describe how a fresh deploy reproduces it deterministically.
4. **Recommended fix wording.** Two viable shapes:
   (a) Remove the `OR unique_identifier IN ?` clause and pass
       `params.IDs` once (eval recommendation).
   (b) Remove the entire `if len(params.IDs) > 0 { ... }` block if
       `id`-based filtering belongs in a different layer.
   **Tentative answer: lead with (a) only**; (b) is architectural and
   not our call.
5. **Cite our eval/evidence paths in the issue body?** Same answer
   as plan-01: yes, as footer "full validation chain" links.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: `1562201` removed the `CommonSSN` embedding from `Datastream` (and with it the `unique_identifier` column). One SQL site in `datastream_repository.go` was not updated and still references the dropped column. |
| **Claim** | One bullet: `?id=` on `/datastreams` returns HTTP 500 on any deployment built fresh from current HEAD because the WHERE clause references a column that no longer exists in the schema. |
| **Static evidence** | Excerpt from `internal/repository/datastream_repository.go` showing the `query.Where("id IN ? OR unique_identifier IN ?", ...)` line, plus a one-line confirmation that the `Datastream` struct (post-`1562201`) carries no `UniqueIdentifier`/`unique_identifier` field. From [`evidence/issue-012/static-analysis-2026-04-30.md`](../evidence/issue-012/static-analysis-2026-04-30.md) §3. |
| **Live evidence** | Three `curl` probes with HTTP 500 responses, plus the additive-AutoMigrate caveat. From [`evidence/issue-012/live-test-2026-04-30.md`](../evidence/issue-012/live-test-2026-04-30.md), refreshed during Step 4.2. |
| **Spec authority** | Single sentence: the canonical CSAPI Part 2 Datastream schema (OGC 23-002) defines no `uid` property on Datastream, supporting the conclusion that the cleanup direction (drop) was correct and the fix is to delete the dangling SQL clause, not re-add a column. |
| **Recommended fix** | Replace `query = query.Where("id IN ? OR unique_identifier IN ?", params.IDs, params.IDs)` with `query = query.Where("id IN ?", params.IDs)`. |
| **Scope guard** | "What NOT to touch": the `Datastream` domain model (correct as-is post-`1562201`); `AutoMigrate`; any other repository's filter logic; the `ControlStream` repository (`unique_identifier` may legitimately exist there if `CommonSSN` is still embedded — out of scope for this issue). |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-02-datastream-dangling-unique-identifier-sql.md`](../upstream-issue-reports/report-02-datastream-dangling-unique-identifier-sql.md)

Same `NN` and `<slug>`
(`02-datastream-dangling-unique-identifier-sql`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
