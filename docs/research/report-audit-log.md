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

---

## report-03-controlstream-systems-json-leak

**Audit date:** 2026-05-05
**Verdict:** `pass` (after in-place citation fixes)
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table)** — backlog/plan paths OK; repo handle / external pre-work URL `N/A`.
- **CHECK-2 (Source fork issue numbers)** — `OS4CSAPI/connected-systems-go#15` referenced 3× (header, §8, §10); all `N/A: gh unavailable`.
- **CHECK-3 (Commit SHAs)** — `df6da0d`, `d2d1347`, `dacae7b`, `f2cf1c3`, `2dc09f7` all **OK** (reachable + ancestors of `upstream/main`).
- **CHECK-4 (Commit messages)** — all 5 quoted subjects match `git log -1 <SHA> --format='%s'` verbatim **OK** (`2dc09f7` quote uses Unicode ellipsis vs. literal `...`; non-substantive).
- **CHECK-5 (File paths)** — `internal/model/domains/control_stream.go`, `internal/model/domains/datastream.go` both **OK**.
- **CHECK-6 (Line numbers)** — no explicit `<file>:<line>` citations; subagent confirmed quoted struct-tag string is at `control_stream.go:52` and `datastream.go:56`. **N/A** for absolute drift; **OK** for content match.
- **CHECK-7 (Code snippets)** — `Systems []System \`gorm:"many2many:system_controlstreams;"\`` (current) and `Systems []System \`gorm:"many2many:system_datastreams;" json:"-"\`` (parent post-fix) match upstream verbatim; `2dc09f7 --stat` confirms only `datastream.go` was touched **OK**; live curl bodies `N/A` (live capture).
- **CHECK-8 (Cross-references)** — initial pass returned 2× FAIL on `static-analysis-2026-04-30.md §3.4` (file has only top-level §3, no `§3.4`); also stale "backlog will be corrected" / "P2 — sibling entry" header line. **All three fixed in-place pre-filing** by replacing `§3.4` → `§3`, restating header severity rationale, and converting backlog-correction language to past tense.
- **CHECK-9 (Spec citations)** — OGC 23-002 (CSAPI Part 2 `controlStream.json` / `baseStream.json`); RFC 7493 §4.3 (I-JSON); JSON Schema 2020-12 `additionalProperties` semantics; OAS 3.0.3 / OGC 23-001 / OGC 19-072 cited as **out-of-scope**.
- **CHECK-10 (Internal consistency §1 vs §10)** — HEAD SHA, file paths, parent-fix `2dc09f7`, field name `Systems`, leaked emission `"Systems": null`, struct-tag strings (current vs. proposed), five FK siblings with `json:"-"` — all **OK** (consistent).

```
SUMMARY (post-fix):
- Checks attempted: 44
- OK: 36
- FAIL: 0 (3 pre-fix FAILs all resolved in-place: §3.4 citation drift ×2, stale header severity language)
- N/A: 8 (3× gh unavailable for #15, repo handle, external URL, line-content-only matches, live curl)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P3-Minor) | **Sound.** Sibling cannot exceed parent triage; parent #15 was P3-Minor. Eval §3.6 explicitly rejected P2 ("functional or strict-spec impact; neither exists"). |
| Framing accuracy | **Sound.** Parent-commit-as-precedent narrative is rigorous: `2dc09f7` shape is mirrored exactly. Issue #15's own "file siblings separately" rule cited as the reason for separate filing. |
| Scope guard (§7) | **Sound.** Single struct-tag append, no logic, no migration. Latent §3.5 hardening explicitly deferred to backlog #6. OSH publisher fleet bootstrap (report-13) noted as out-of-scope. |
| Recommended fix realism | **Sound.** One-line `json:"-"` append. Zero risk by precedent — same change on parent struct merged cleanly. |
| Public extract self-containment | **Sound.** §10 stands alone — parent commit reference, side-by-side struct-tag comparison, live curl of both controlstream (with leak) and datastream (without) on the same deployment, OGC 23-002 + RFC 7493 spec authority, one-line fix. |
| Companion-report cross-references | Parent #15 (closed, fork-side) cited; backlog #6 (latent slices) explicitly out-of-scope; report-13 (OSH publisher) noted as not-yet-affected. All resolve. |

### Action items
- Pre-filing fixes applied in-place: `§3.4` → `§3` (×2), header severity language past-tense, §8 backlog-status past-tense, §9 question-row past-tense.

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/3>
- **Filed:** 2026-05-05

---

---

## report-04-systemevent-not-in-deletecascade

**Audit date:** 2026-05-05
**Verdict:** `pass`
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table)** — backlog/plan paths OK; repo handle / external pre-work URL `N/A`.
- **CHECK-2 (Source fork issue numbers)** — `OS4CSAPI/connected-systems-go#16` referenced 3× (header, §1 L73, §8); all `N/A: gh unavailable`.
- **CHECK-3 (Commit SHAs)** — `df6da0d`, `fe9fbd0`, `c2ab201`, `1b2b614`, `e0f31c4`, `5b5fb94`, `dacae7b` all **OK** (reachable + ancestors of `upstream/main`).
- **CHECK-4 (Commit messages)** — all 7 quoted subjects match `git log -1 <SHA> --format='%s'` verbatim **OK** (ellipsis-vs-three-dots non-substantive).
- **CHECK-5 (File paths)** — `internal/repository/system_repository.go`, `internal/model/domains/system_event.go`, `internal/repository/system_event_repository.go` all **OK**.
- **CHECK-6/CHECK-7 (Code snippets)** — `deleteCascade` Select-String block (recursive Systems, SamplingFeature parent_system_id, deleteSystemDatastreams, deleteSystemControlStreams, SystemHistoryRevision, system_deployments, system_procedures, final `tx.Delete(&domains.System{}, "id = ?", systemID)`) all verified present **OK**; `SystemEvent` / `system_events` zero-match in `system_repository.go` reproduced **OK**; `SystemEvent` struct snippet (`Base`, `SystemID string \`gorm:"type:varchar(255);index;not null" json:"-"\``) matches upstream byte-for-byte **OK**; 6-line `git log` of `system_repository.go` matches **OK**.
- **CHECK-8 (Cross-references)** — `../upstream-followup-backlog.md`, `../upstream-issues/plan-04-...md`, `../evidence/issue-002/fk-constraints-head-2026-04-30.txt`, `../evidence/issue-016/live-test-2026-04-30.md`, `../evidence/issue-016/static-analysis-2026-04-30.md`, `../issue-evaluations/issue-016.md` all resolve **OK**; external pre-work URL `N/A`.
- **CHECK-9 (Spec citations enumerated, not validated)** — OGC 23-001 (CSAPI Part 1 §"System events"); OGC 23-002 (CSAPI Part 2 SystemEvent schema); OAS 3.0.3 / OGC 19-072 / RFC 7493 cited as **out-of-scope**.
- **CHECK-10 (Internal consistency §1 vs §10)** — HEAD SHA, pivotal `fe9fbd0`, file paths, `SystemID` gorm tag, child-enumeration count (7 present + SystemEvent missing = expected 8), recommended-fix snippet, P2 severity — all **OK** (consistent).

```
SUMMARY:
- Checks attempted: 50
- OK: 44
- FAIL: 0
- N/A: 6 (3× gh unavailable for #16, repo handle, external URL, deferred CHECK-1)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P2) | **Sound.** Silent integrity loss on publicly invokable cascade-delete path; no 5xx, no client signal. P1 reserved for active-data-corruption-on-write or auth-bypass; P3 too soft for silent orphan on every cascade delete. |
| Framing accuracy | **Sound.** "Completing 3b" framing is precedent-bound: `fe9fbd0` adopted Approach 3b, this filing finishes the enumeration. Cites maintainer's own merged commit as the model. Approach 3a (FK constraints) deferred to backlog #14 with one-line mention. |
| Scope guard (§7) | **Sound.** Carves out schema migration, FK addition, Approach 3a vs 3b re-litigation, non-cascade `Delete` branch (rejected ErrHasChildren on grounds SystemEvent is value-class child). §1 audit confirms SystemEvent is sole gap. |
| Recommended fix realism | **Sound.** Three-line append mirroring `SystemHistoryRevision`. Zero-risk by construction; same shape as 7 sibling deletes already merged. |
| Public extract self-containment | **Sound.** §10 stands alone — `fe9fbd0` Approach 3b context, deleteCascade enumeration with the gap, struct field with `not null` no-FK, recommended fix mirroring sibling, OGC 23-001 + 23-002 spec authority, P2 severity rationale. |
| Companion-report cross-references | Parent #16 (closed) cited; backlog #14 (Approach 3a) deferred. Issue #2 evidence pack (FK constraints) cross-linked for the no-FK claim. All resolve. |

### Action items
None.

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/4>
- **Filed:** 2026-05-05

---

---

## report-06-totimerange-year-0001-silent-discard

**Audit date:** 2026-05-05
**Verdict:** `pass` (after in-place line-range fixes)
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table)** — backlog/plan paths OK; external pre-work URL `N/A`.
- **CHECK-2 (Source fork issue numbers)** — `OS4CSAPI/connected-systems-go#18` referenced 3× (header, §8, §10); all `N/A: gh unavailable`.
- **CHECK-3 (Commit SHAs)** — `df6da0d`, `c2ab201`, `1b2b614`, `554ada5`, `6b856bb` all **OK** (reachable + ancestors of `upstream/main`).
- **CHECK-4 (Commit messages)** — all 5 quoted subjects match `git log -1 <SHA> --format='%s'` verbatim **OK**.
- **CHECK-5 (File paths)** — `internal/model/common_shared/time_range.go`, `history.go`, `time_range_test.go` all **OK**.
- **CHECK-6 (Line numbers)** — point-line citations (`time_range.go:54/119/129/147/152/161/298/314`, `history.go:39`, `time_range_test.go:112`) all **OK**; `UnmarshalJSON` array branch "73-89" / object branch "98-115" within fuzz **OK**; `toTimeRangeStrict` line 215 **OK**. Two function-range citations FAILed pre-fix: `ToTimeRange "lines 129-170"` (actual 129-166, +4 drift) and `ToTimeRangeFromSlice "lines 187-201"` (actual 172-194, range starts at second guard and runs into next function). **Both fixed in-place pre-filing** (129-170 → 129-166 ×1; 187-201 → 172-194 ×2).
- **CHECK-7 (Code snippets)** — `ToTimeRange` body, `UnmarshalJSON` string-form, `history.go:39` excerpt, `ParseTimeRange(string)` adapter, `ParseTimeRange(map)` adapter, §10 `ToTimeRangeFromSlice` correct-guard snippet — all match upstream verbatim **OK**.
- **CHECK-8 (Cross-references)** — `../upstream-followup-backlog.md` #9 (heading verified at line 122), `../upstream-issues/plan-06-...md`, `../evidence/issue-018/static-analysis-2026-04-30.md`, `live-test-2026-04-30.md`, `spec-authority-2026-04-30.md`, `../issue-evaluations/issue-018.md` §"Adjacent finding" (heading verified at line 42) all **OK**.
- **CHECK-9 (Spec citations enumerated, not validated)** — RFC 7493 §3.4 (I-JSON); OGC 23-001 (CSAPI Part 1 `phenomenonTime`); OGC 23-002 (CSAPI Part 2 Datastream/Observation schemas); RFC 9110 §15.5.1 (400 Bad Request); OAS 3.0.3 / JSON Schema 2020-12 cited as **out-of-scope**.
- **CHECK-10 (Internal consistency §1 vs §10)** — HEAD SHA, three legacy-site lines (147/152/161), four-site caller graph, parent fix `c2ab201`, file paths, P3 severity — all **OK** (consistent).

```
SUMMARY (post-fix):
- Checks attempted: 49
- OK: 43
- FAIL: 0 (2 pre-fix line-range FAILs all resolved in-place)
- N/A: 6 (3× gh unavailable for #18, external URL, repo handle, subjective deferral)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P3) | **Sound.** Parent `c2ab201` covers high-traffic UnmarshalJSON array/object branches; residual surface is slash-delimited string form (less canonical) + `history.go` HistoricTime fallback. P2 reserved for parent. |
| Framing accuracy | **Sound.** "Residual of c2ab201 / #18" framing is precedent-bound and explicit. Maintainer's own `c2ab201` introduced the new single-value legacy site (legacy site #3) — confirms institutional pattern, not one-off. |
| Scope guard (§7) | **Sound.** Carves out UnmarshalJSON array/object (already strict), Observation.ResultTime, ToTimeRange removal, new API surface (toTimeRangeStrict already exists). Audit confirms only two files + internal adapters touched. |
| Recommended fix realism | **Sound.** Three guards in one function, no API change, no caller migration. Mirrors neighbouring `ToTimeRangeFromSlice` exactly. Options B/C (structural strict-parse) offered as opt-in PR. |
| Public extract self-containment | **Sound.** §10 stands alone — `c2ab201` precedent, four-site caller graph, before-state code, sibling correct-guard reference, three-guard diff fix, RFC 7493 + OGC 23-001 + RFC 9110 spec authority, opt-in structural alternative. |
| Companion-report cross-references | Parent #18 cited; backlog #9 cross-linked. No companion reports. |

### Action items
- Pre-filing fixes: §2 "lines 129-170" → "129-166"; §6 + §10 "lines 187-201" → "lines 172-194" (×2 occurrences).

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/5>
- **Filed:** 2026-05-05

---

---

## report-07-samplingfeature-id-silent-drop

**Audit date:** 2026-05-05
**Verdict:** `pass`
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table)** — backlog #10 / plan paths OK (heading `### 10. samplingFeature@id retains silent-drop type assertion` confirmed); fork-issue/repo handle `N/A`.
- **CHECK-2 (Source fork issue numbers)** — `OS4CSAPI/connected-systems-go#19` referenced 4× (header, §8, §10 multiple); all `N/A: gh unavailable`.
- **CHECK-3 (Commit SHAs)** — `df6da0d`, `1b2b614` both **OK** (reachable + ancestors of `upstream/main`).
- **CHECK-4 (Commit messages)** — `code sight updates`, `Adding support for "latest" for TimeRange` — both match verbatim **OK**.
- **CHECK-5 (File paths)** — `internal/api/observation_handler.go`, `internal/api/command_handler.go` both **OK**.
- **CHECK-6 (Line numbers)** — point citations `command_handler.go:217/221/225/229`, `observation_handler.go:225/238/252` all **OK** at exact lines; range `observation_handler.go:238-263` within fuzz **OK**.
- **CHECK-7 (Code snippets)** — bundle-grep output, `decodeObservationPayload` Select-String block (samplingFeature@id legacy + resultTime/phenomenonTime strict-typed with verbatim error messages), `decodeCommandPayload` snippet, post-1b2b614 RFC3339 parse block, §10 legacy + post-fix snippets — all match upstream verbatim **OK**; recommended-fix diffs (minus-side matches upstream, plus-side proposal).
- **CHECK-8 (Cross-references)** — plan-07, backlog-#10, evidence/issue-019/{static,live,spec}-2026-04-30, issue-evaluations/issue-019.md §"Adjacent finding" (heading verified) all **OK**; external pre-work URL `N/A`.
- **CHECK-9 (Spec citations enumerated, not validated)** — OGC 23-001 (CSAPI Part 1 §5.1 / Observation schema); OGC 23-002 (CSAPI Part 2 Observation/Command schemas); RFC 7493 §3.4 (I-JSON wrong-type rejection); RFC 9110 §15.5.1 (400 Bad Request); OAS 3.0.3 / JSON Schema 2020-12 / SWE Common cited as **out-of-scope**.
- **CHECK-10 (Internal consistency §1 vs §10)** — HEAD SHA, parent fix `1b2b614`, file paths, line numbers (`observation_handler.go:225/238/252`, `command_handler.go:217`), field names (`samplingFeature@id`, `SamplingFeatureID`, `phenomenonTime`, `resultTime`), P3 severity — all **OK** (consistent).

```
SUMMARY:
- Checks attempted: 49
- OK: 43
- FAIL: 0
- N/A: 6 (gh unavailable for #19, external pre-work URL, repo handle)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P3) | **Sound.** §9 Q3 explicitly considered P2 vs P3: chose P3 because no repository default-fill exists for `samplingFeature@id`, so silent-drop surfaces as client-visible null in GET response (vs. parent #19 where default-fill could produce a plausibly-correct substitution). Narrower blast radius than parent. |
| Framing accuracy | **Sound.** "Residual of 1b2b614" framing is precedent-bound — same decoder, same function, same package. Fix is byte-for-byte mirror of parent's pattern. Asymmetry-within-same-function presentation is unusually tight. |
| Scope guard (§7) | **Sound.** Bundles `command_handler.go:217` sibling (same field, same shape, same package) per plan §8 Q2; explicitly does NOT bundle other `command_handler.go` legacy patterns (`sender`, `currentStatus`, `issueTime`, `executionTime` — different fields). Rationale documented. |
| Recommended fix realism | **Sound.** Two block edits, byte-for-byte mirror of merged precedent. No new types, no API change. Diff shown for one site; "and analogous in `command_handler.go`" for the second. |
| Public extract self-containment | **Sound.** §10 stands alone — `1b2b614` precedent, side-by-side asymmetry within `decodeObservationPayload`, sibling site in `command_handler.go`, two-file fix diff, OGC 23-001 + 23-002 + RFC 7493 + RFC 9110 spec authority, P3 rationale. |
| Companion-report cross-references | Parent #19 cited; backlog #10 cross-linked. No companions. |

### Action items
None.

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/6>
- **Filed:** 2026-05-05

---

---

## report-08-empty-string-resulttime-phenomenontime

**Audit date:** 2026-05-05
**Verdict:** `pass`
**Filing-ready:** yes

### §2.1 Mechanical findings (subagent, verbatim)

Pre-flight: `git log -1 upstream/main --format='%H'` → `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` — matches stated audit reference HEAD. ✓

- **CHECK-1 (Header table)** — backlog #11 (heading `### 11. resultTime / phenomenonTime empty-string conflates with missing` confirmed at L148), plan-08 path **OK**; external pre-work URL and upstream repo identifier `N/A`.
- **CHECK-2 (Source fork issue numbers)** — `OS4CSAPI/connected-systems-go#20` referenced in header, §2, §8, §10; all `N/A: gh unavailable` (logged for manual spot-check).
- **CHECK-3 (Commit SHAs)** — HEAD `df6da0d…`, parent `1b2b614`, plus `fe9fbd0`, `635547f`, `f2cf1c3` (in §1 log preview) — all **OK** (reachable + ancestors).
- **CHECK-4 (Commit messages)** — `code sight updates`, `Adding support for "latest" for TimeRange`, `Adding cascade delete and fixing existing cascade delete to full delete`, `making the default limit for pagination configurable`, `Adding the other resources along with e2e tests` — all match verbatim **OK**.
- **CHECK-5 (File paths)** — `internal/api/observation_handler.go` cited in §1, §2, §6, §10 — **OK**.
- **CHECK-6 (Line numbers)** — report does not cite `<file>:<line>` numbers (block-level citations only) — `N/A`.
- **CHECK-7 (Code snippets)** — `resultTime` decoder block (rtRaw/ok/`if rtStr != ""`/parse/assign with verbatim error message), `phenomenonTime` analogous block with `obs.PhenomenonTime = &t` pointer assignment, late `obs.ResultTime.IsZero()` guard with `"resultTime is required"` message, behavior matrix observable from code shape, claimed-zero-match grep for `rtStr == ""|ptStr == ""|must be a non-empty` — all **OK** (re-verified; zero matches confirmed).
- **CHECK-8 (Cross-references)** — evidence/issue-020/{static,live,spec}-2026-04-30, issue-evaluations/issue-020.md, plan-08, backlog #11 — all **OK**; external pre-work URL `N/A`.
- **CHECK-9 (Spec citations enumerated, not validated)** — RFC 7807 §3 (primary, problem-detail accuracy); OGC 23-001 §Error responses; OGC 19-072 §error responses (OGC API – Common Part 1); RFC 9110 §15.5.1 (400 Bad Request); OAS 3.0.3 / JSON Schema 2020-12 / RFC 7493 cited as **out-of-scope**.
- **CHECK-10 (Internal consistency §1 vs §10)** — HEAD SHA, parent fix `1b2b614`, file path, function name `decodeObservationPayload`, identifiers (`rtRaw/rtStr/ptRaw/ptStr`), pointer assignment (`obs.PhenomenonTime = &t`), late guard (`obs.ResultTime.IsZero()`), behavior matrix rows, severity P3 — all **OK** (consistent).

```
SUMMARY:
- Checks attempted: 38
- OK: 30
- FAIL: 0
- N/A: 8 (gh unavailable for #20 across multiple references; external pre-work URL; external upstream repo identifier; CHECK-6 not applicable)
- Critical concerns: none
```

### §2.2 Subjective findings (orchestrator)

| Axis | Finding |
|---|---|
| Severity calibration (P3) | **Sound.** §9 Q4 explicitly weighed P2: phenomenonTime silent-accept-empty is "data-integrity-adjacent" but PhenomenonTime is optional in CSAPI Part 1, so empty-string→nil is semantically equivalent to field omission. Pure problem-detail accuracy + asymmetric-message UX. P3 holds. |
| Framing accuracy | **Sound.** "Last gap of 1b2b614 matrix" framing is precedent-bound — body of #20 explicitly proposed the empty-string branch the shipped commit omitted. Asymmetric-symptoms framing (one decoder shape, two surfaces) is unusually crisp and surfaces a genuine sibling defect not in the original matrix. |
| Scope guard (§7) | **Sound.** Excludes wrong-type/missing/null branches (covered by `1b2b614`), late `IsZero()` guard (retained as defense-in-depth), null↔missing collapse (out of scope per maintainer's accepted framing in #20), `samplingFeature@id`/`command_handler.go`/`ToTimeRange` (separate filings — report-07, report-06). Tight. |
| Recommended fix realism | **Sound.** Two block edits, one function. Mirrors maintainer's own #20-body proposal byte-for-byte. Diff shown for one site; "analogous edit" for the second. No new types, no API change. |
| Public extract self-containment | **Sound.** §10 stands alone — `1b2b614` precedent, asymmetric-symptoms behavior matrix, full code-shape excerpt (resultTime + phenomenonTime + late guard), zero-match grep, fix diff, RFC 7807 + OGC 23-001 + OGC 19-072 + RFC 9110 spec authority, P3 rationale with phenomenonTime optional-field reasoning. |
| Companion-report cross-references | Parent #20 cited in §8 + §10. Report-06 (`ToTimeRange` family) and report-07 (`samplingFeature@id`) cross-referenced in §7 scope guard. No fork-side companions. |

### Action items
None.

### Filed
- **Upstream issue:** <https://github.com/SomethingCreativeStudios/connected-systems-go/issues/7>
- **Filed:** 2026-05-05

---
