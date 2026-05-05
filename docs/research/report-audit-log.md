# Report Audit Log

**Audit reference state:**

- `upstream/main` HEAD at audit kickoff: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` ("code sight updates")
- Audit kickoff date: 2026-05-05
- Filing cadence: file each `pass`-verdict report immediately
- `gh` CLI: **unavailable on PATH** — fork-issue (`#NN`) existence checks dispatched to subagent will be marked `N/A: gh unavailable` and spot-checked manually via browser only if the citation is critical.

Ordering (per [report-audit-plan.md §5](./report-audit-plan.md)):
01 → 02 → 03 → 04 → 06 → 07 → 08 → 09 → 10 → 11 → 12 → 13 → backlog.

---

## report-01-api-endpoint-stub

**Audit date:** 2026-05-05
**Verdict:** `pass`
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table paths)** — all in-repo paths OK; `OS4CSAPI/connected-systems-go#14`, `SomethingCreativeStudios/connected-systems-go`, comment ID `4380617500` marked `N/A: gh unavailable`.
- **CHECK-2 (Source fork issue numbers)** — `#14` referenced 3× (L22, L283, L424); all `N/A: gh unavailable`.
- **CHECK-3 (Commit SHAs)** — `df6da0d` / `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (L34, L42, L255) **OK**; `2dc09f7` (prior HEAD, L34) **OK** (resolves to `2dc09f78178b613faed78285da980569cd0a33f2`, ancestor of upstream/main).
- **CHECK-4 (Commit messages)** — `df6da0d` subject `"code sight updates"` **OK**.
- **CHECK-5 (File paths)** — `internal/api/router.go`, `internal/api/landing_handler.go`, `internal/api/conformance_handler.go` all **OK**.
- **CHECK-6 (Line numbers)** — `internal/api/conformance_handler.go:26` cited as "fully wired" reference: line 26 is the `// GetConformance returns ...` doc-comment, function definition is line 27 (drift +1, within ±2 fuzz tolerance, **OK**).
- **CHECK-7 (Code snippets)** — §1, §2, and §10 code blocks from `router.go` and `landing_handler.go` all **OK**; `git grep go:embed|openapi.*\.(yaml|json)` zero-match reproduced **OK**; live curl response (88 bytes, `application/vnd.oai.openapi+json;version=3.0`) `N/A` (live HTTP not reproducible from source, but body shape is consistent with `getOpenAPISpec` template).
- **CHECK-8 (Cross-references)** — all relative links to `../upstream-issues/plan-01-...md`, `../upstream-followup-backlog.md`, `../evidence/issue-014/...` resolve **OK**; external curated-references URL `N/A: out of scope`.
- **CHECK-9 (Spec citations enumerated, not validated)** — OAS 3.0.3 §4.7.1.1; OGC 19-072 `/req/landing-page/api-definition-success` clauses B + C; OGC 19-072 `/rec/landing-page/api-definition-oas`; `/req/oas30/*` (cited as not-binding); OGC 23-001/23-002 (cited as out-of-scope); media type `application/vnd.oai.openapi+json;version=3.0`.
- **CHECK-10 (Internal consistency §1 vs §10)** — SHA, file paths, quoted comment, body length/shape, Content-Type, conformance-class binding, spec rules — all **OK** (identical between §1 and §10).

```
SUMMARY:
- Checks attempted: 41
- OK: 31
- FAIL: 0
- N/A: 10 (all gh-unavailable, external URL, or live-HTTP)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P2) | **Sound.** §9 Q2 explicitly considered P1 vs P2; chose conservative on grounds that P1 is reserved for formal certification / automated client tooling roadmaps. Binding `SHALL` violation + violation of server's own self-asserted OAS 3.0 schema = clear P2. |
| Framing accuracy | **Sound.** Anchored to verified static evidence (`getOpenAPISpec` literal) + verified `/conformance` declarations. Precision-scoping of `/req/oas30/*` as **not binding** (because `/conf/oas30` is not declared) is exactly right and captured in §4. |
| Scope guard (§7) | **Sound.** Carves out landing-page handler, `/conformance` endpoint, JSON encoder, `/conf/oas30` declaration decision. |
| Recommended fix realism | **Sound.** Three options ranked with rationale; OAS 3.0→3.1 media-type shift surfaced explicitly as a side-effect. **Option 3 premise verified independently:** Part 1 + Part 2 OpenAPI bundles exist at `schemas.opengis.net/ogcapi/connected-systems/part{1,2}/1.0/openapi/openapi-connectedsystems-{1,2}.yaml` (last modified 2025-07-15); both are `openapi: 3.1.0`, modular structure with `paths/parameters/requests/responses/schemas/` subdirs, version `0.0.1` (draft). |
| Public extract self-containment | **Sound.** §10 stands alone — live curl response, static handler code, spec citations with binding/non-binding distinctions, ranked recommendation, validation-chain footer. |
| Companion-report cross-references | N/A (no companions). |

### Action items
None.

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/1>
- **Filed:** 2026-05-05

---

---

## report-02-datastream-dangling-unique-identifier-sql

**Audit date:** 2026-05-05
**Verdict:** `pass`
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table paths)** — all in-repo paths OK; `SomethingCreativeStudios/connected-systems-go` is a repo handle.
- **CHECK-2 (Source fork issue numbers)** — `OS4CSAPI/connected-systems-go#12` referenced 3× (L20, L217, ~L376); all `N/A: gh unavailable`.
- **CHECK-3 (Commit SHAs)** — `df6da0d`, `2dc09f7`, `d2d1347`, `1562201`, `dacae7b`, `f2cf1c3` all **OK** (reachable + ancestor of `upstream/main`).
- **CHECK-4 (Commit messages)** — all 6 quoted subjects match `git log -1 <SHA> --format='%s'` verbatim **OK**.
- **CHECK-5 (File paths)** — `internal/repository/datastream_repository.go`, `internal/model/domains/datastream.go` both **OK**.
- **CHECK-6 (Line numbers)** — report uses `Select-String -Context` blocks without absolute line citations; **N/A** (no explicit `<file>:<line>` references to validate).
- **CHECK-7 (Code snippets)** — `applyFilters` signature + `Where("id IN ? OR unique_identifier IN ?", ...)` matches upstream verbatim in §1, §2, §10; `git grep` zero-match for `CommonSSN|UniqueIdentifier|unique_identifier` in `datastream.go` reproduced **OK**; `git log --oneline ... -- datastream.go | head -5` matches the reported 5-line list verbatim **OK**.
- **CHECK-8 (Cross-references)** — `../upstream-followup-backlog.md` (#4), `../upstream-issues/plan-02-...md`, `../evidence/issue-012/static-analysis-2026-04-30.md`, `../evidence/issue-012/live-test-2026-04-30.md`, `docs/research/issue-evaluations/issue-012.md` all **OK**; external pre-work URL `N/A: out of scope`.
- **CHECK-9 (Spec citations enumerated, not validated)** — OGC 23-002 (CSAPI Part 2 Datastream schema, `baseStream.json` / `dataStream.json`); OAS 3.0.3 / OGC 19-072 / OGC 23-001 cited as **out-of-scope**.
- **CHECK-10 (Internal consistency §1 vs §10)** — HEAD SHA, file paths, `1562201` framing as removal point, `applyFilters` snippet text, `git grep` no-matches + 5-line log, OGC 23-002 spec citation — all **OK** (identical between §1, §2, and §10).

```
SUMMARY:
- Checks attempted: 36
- OK: 28
- FAIL: 0
- N/A: 8 (3× gh unavailable for #12, repo handle, external URL, no explicit line-number citations, pre-work URL)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P2) | **Sound.** §9 Q2 confirmed P2 over P3-latent: documented filter parameter returns 500 deterministically on fresh deploys. Plan's backlog-accuracy flag corroborated. |
| Framing accuracy | **Sound.** Cleanup-miss framing with HTTP-500 impact front-loaded. `1562201`-as-the-removal-commit is verified. AutoMigrate caveat (§3 + §10) preempts dismissal-by-stale-dev-DB. |
| Scope guard (§7) | **Sound.** Carves out domain model, AutoMigrate, other repos' filter logic, ControlStream sibling, `uid` field re-introduction. ControlStream cross-reference points correctly to backlog #5 / report-03. |
| Recommended fix realism | **Sound.** One-line removal of the `OR unique_identifier IN ?` clause. Zero-risk by construction (column does not exist in post-`1562201` schema). Option B (drop entire `if` block) explicitly rejected as architectural / not-our-call. |
| Public extract self-containment | **Sound.** §10 stands alone — `1562201` reference, `applyFilters` snippet, `git grep` + `git log` evidence, three live curl probes with caveat, spec authority (OGC 23-002 no-`uid`), one-line fix, scope guard, validation chain. |
| Companion-report cross-references | ControlStream sibling deferred to report-03 / backlog #5. Other Adjacent Findings B (DELETE 500) and C (3 orphaned test datastreams) explicitly logged as out-of-scope for this filing. |

### Action items
None.

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/2>
- **Filed:** 2026-05-05

---
