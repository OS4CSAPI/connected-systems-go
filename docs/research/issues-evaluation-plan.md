# Issues Evaluation Plan — `OS4CSAPI/connected-systems-go`

**Purpose.** Conduct a deep, thorough, comprehensive evaluation of every open issue
in our `connected-systems-go` fork to determine each issue's **validity,
legitimacy, accuracy, and completeness** before any of them are acted on,
escalated, or closed.

**Why this plan exists.** The issue tracker is the single source of truth that
will drive future development priority. Issues filed in haste, derived from
incomplete code reads, or based on misidentified failure modes will silently
distort that priority. Investing the time to audit each issue once — methodically
and with citations — pays back across every downstream decision.

**Scope.** All issues open at the start of this effort on
<https://github.com/OS4CSAPI/connected-systems-go/issues>. PRs are out of scope
for this plan. New issues filed after the start date are handled as a follow-up.

**Non-goals.** This plan does **not** implement fixes, propose patches, or
re-prioritize roadmap items. Those are downstream activities. The deliverable is
a per-issue evaluation record only.

---

## Working agreement

1. **One issue at a time.** No batch evaluations. Each issue gets a dedicated
   evaluation pass and a dedicated record.
2. **Cite, don't claim.** Every finding-of-fact in an evaluation must reference
   either (a) a specific file:line in the cs-go source tree, (b) an authoritative
   external standard (OGC API, WHATWG, RFC, IETF), or (c) a verbatim quote from
   the issue body. Unsourced assertions are not acceptable.
3. **Distinguish layers of confidence.** Use the rubric below. Do not collapse
   "the issue describes a real symptom" with "the issue's diagnosis is correct"
   with "the issue's proposed fix is appropriate." Those are three separate
   judgments.
4. **Admit uncertainty.** If an evaluation cannot reach a conclusion in a given
   dimension without further code execution, server access, or upstream
   clarification, the record says so explicitly. We do not invent confidence.
5. **No issue is closed or commented on GitHub** as part of this effort without
   explicit user approval per issue. The plan produces *recommendations*; acting
   on them is a separate decision.

---

## Authoritative resources

The sibling planning repo `OS4CSAPI/ogc-client-CSAPI_2` (branch `phase-7`) has
an extensive `docs/research/` tree of pre-digested CSAPI specification material
and authoritative references. **Spec-related verdicts in our evaluations must
cite one of these resources or the upstream standard directly.** "I think the
spec says…" is not acceptable.

### Primary (machine-readable, authoritative)

- **OGC API – Connected Systems Part 1, bundled OpenAPI 3.1** —
  [`docs/research/standards/ogcapi-connectedsystems-1.bundled.oas31.yaml`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/standards/ogcapi-connectedsystems-1.bundled.oas31.yaml)
  · all `$ref`s resolved · use for endpoint paths, request/response schemas,
  query parameters, conformance classes for Systems / Deployments / Procedures /
  SamplingFeatures / Properties / Features.
- **OGC API – Connected Systems Part 2, bundled OpenAPI 3.1** —
  [`docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml)
  · all `$ref`s resolved · use for Datastreams / Observations / ControlStreams /
  Commands / SystemEvents / SystemHistory.

### Pre-digested spec extractions (use these to find the right OpenAPI region)

- [`docs/research/requirements/csapi-part1-requirements.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-part1-requirements.md)
- [`docs/research/requirements/csapi-part2-requirements.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-part2-requirements.md)
- [`docs/research/requirements/csapi-crud-operations.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-crud-operations.md)
  — directly relevant to issues #1, #2, #7, #12.
- [`docs/research/requirements/csapi-query-parameters.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-query-parameters.md)
  — directly relevant to issues #7, #8, #9, #11.
- [`docs/research/requirements/csapi-format-requirements.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-format-requirements.md)
  — directly relevant to issues #3, #4, #5.
- [`docs/research/requirements/csapi-datatype-schema-requirements.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-datatype-schema-requirements.md)
- [`docs/research/requirements/csapi-conformance-capabilities.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-conformance-capabilities.md)
- [`docs/research/requirements/csapi-gap-analysis.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-gap-analysis.md)
- [`docs/research/requirements/csapi-oshconnect-python-analysis.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-oshconnect-python-analysis.md)
  — the consumer that surfaces in many cs-go issue bodies.
- [`docs/research/requirements/csapi-opensensorhub-analysis.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/requirements/csapi-opensensorhub-analysis.md)

### Index of external standards

- [`docs/research/references.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/references.md)
  — annotated bibliography: OGC 23-001, OGC 23-002, SensorML 3.0, SWE Common 3.0,
  related OGC APIs, foundational semantic standards.

### How to use

For each issue evaluation:

1. Identify which CSAPI resource(s) the issue concerns (Datastream? Observation?
   Deployment? query parameters? schema validation?).
2. Open the relevant pre-digested file above to find the right OpenAPI region.
3. **Verify against the bundled OpenAPI directly** — quote the schema verbatim
   in the evaluation record's Evidence block.
4. If a verdict turns on a claim that is not in the OpenAPIs (e.g. SensorML
   semantics, SWE Common encoding details), use `references.md` to find the
   authoritative external standard URL.

**Do not copy these files into the cs-go repo.** They are large and live
canonically in the planning repo; reference by URL and quote the relevant
fragment in the evaluation record.

---

## Evaluation rubric

For each issue we record a verdict on four dimensions, plus an overall
recommendation.

| Dimension | Question | Possible verdicts |
|---|---|---|
| **Validity** | Does the issue describe a real, reproducible problem in the current `main` branch of cs-go? | `confirmed` / `partially-confirmed` / `not-reproducible` / `cannot-determine` |
| **Legitimacy** | Is the described behavior actually wrong relative to the OGC API – Connected Systems specifications and/or stated cs-go design intent? Or is it expected/by-design? | `defect` / `spec-gap-in-cs-go` / `working-as-designed` / `out-of-scope-for-cs-go` / `cannot-determine` |
| **Accuracy** | Are the technical claims in the issue body (file paths, line numbers, function names, error messages, root cause) factually correct? | `accurate` / `mostly-accurate` / `inaccurate-but-symptom-real` / `inaccurate` |
| **Completeness** | Does the issue contain enough information for a downstream contributor to reproduce, scope, and fix without further investigation? | `complete` / `needs-repro-steps` / `needs-spec-citation` / `needs-scope-clarification` / `incomplete` |

### Overall recommendation

One of:

- **Keep / accept as written** — solid issue, ready for triage.
- **Keep with edits** — real issue but body needs corrections (we list them).
- **Reframe** — symptom is real but diagnosis or scope is wrong; rewrite needed.
- **Convert** — should become a different artifact (research note, doc PR,
  upstream-OGC issue, downstream-consumer issue).
- **Close** — not a defect / out of scope / duplicate.
- **Defer pending information** — cannot judge without X.

---

## Methodology (per-issue procedure)

For each issue:

1. **Read the issue body verbatim.** Capture title, labels, body text, comments,
   and linked artifacts. Do not summarize yet.
2. **Identify load-bearing claims.** Enumerate every factual claim the issue
   makes that the verdict depends on (e.g. "function X at file:line does Y",
   "spec section Z requires W", "request R produces response S").
3. **Verify each claim against the source tree** at the cs-go commit current at
   the time of evaluation. Record commit SHA in the evaluation record.
4. **Verify spec claims against the authoritative document.** OGC API – Connected
   Systems Part 1 / Part 2, RFCs cited in those specs, etc. Use direct quotes
   with section references.
5. **Attempt reproduction** when feasible (against `https://129-80-248-53.sslip.io/csapi-go`
   or a local cs-go instance). If not feasible, record why.
6. **Render verdicts** on the four rubric dimensions and an overall
   recommendation. Show the reasoning, not just the verdict.
7. **Record the evaluation** as `docs/research/issue-evaluations/issue-NNN.md`
   following the template below.
8. **Update this plan's tracking table.**
9. **Stop.** Do not proceed to the next issue without user direction.

### Evaluation record template

Each per-issue file contains the sections below. The structure is adapted from
the `phase-6/findings-report-template.md` pattern in the sibling planning repo
and requires explicit Evidence blocks per claim — unsourced reasoning is not
permitted.

```markdown
# Issue #NNN — <title>

- **URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/NNN
- **State at evaluation:** open|closed
- **Labels:** ...
- **Filed by:** <user>
- **Filed:** YYYY-MM-DD
- **Evaluated against cs-go HEAD:** <full-SHA>
- **Date of evaluation:** YYYY-MM-DD

## Sources Consulted

### Primary (authoritative)
- [OGC bundled OpenAPI / cs-go file:line / RFC / etc.] — what was found

### Supporting
- [requirements doc / external link / etc.] — what was used

## 1. Issue body — load-bearing claims

Enumerate every factual claim the verdict depends on, in a table. Do not
paraphrase reasoning; list claims atomically (C1, C2, C3, …) so each can be
verified independently.

## 2. Verification

For each load-bearing claim, apply the per-claim block:

### 2.X — <Claim header>

**Claim (verbatim or close paraphrase):** ...

**Evidence:**

```
<verbatim code, schema fragment, or spec quote with file:line or section ref>
```

**Analysis:** [one paragraph: does the evidence support, refute, or partially
support the claim? what subtlety matters?]

## 3. Reproduction

What was attempted, what succeeded, what was skipped and why. If skipped,
record under §6 Open questions.

## 4. Verdicts

| Dimension | Verdict | Rationale (one sentence) |
|---|---|---|
| Validity | ... | ... |
| Legitimacy | ... | ... |
| Accuracy | ... | ... |
| Completeness | ... | ... |

## 5. Reasoning summary

2–4 paragraphs synthesising the verdicts. May reference §2 evidence by section
number.

## 6. Recommendation

One of the six standard recommendations from the rubric, with concrete edits if
applicable.

## 7. Open questions / unknowns

What the evaluation could not resolve, and what would be needed to resolve it.
```

---

## Issue inventory & tracking

Snapshot taken at start of effort. State and counts will drift; the per-issue
records are the source of truth as evaluations land.

| # | State | Title (verbatim) | Eval status | Verdict | Record |
|---|---|---|---|---|---|
| 1 | open | Datastream creation without explicit `uid` stores empty string, violates unique constraint on second create | complete | Validity: partially-confirmed · Legitimacy: defect · Accuracy: accurate (stale) · Completeness: complete · **Rec: Keep with edits** | [issue-001.md](issue-evaluations/issue-001.md) |
| 2 | open | DELETE on parent resources fails with raw PostgreSQL FK constraint error instead of cascading or returning structured error | complete | Validity: confirmed (symptom) / partially-refuted (causal model) · Legitimacy: defect (compound — cascade impl is itself broken) · Accuracy: accurate-symptom / inaccurate-mechanism · Completeness: incomplete (does not mention `?cascade=true` partial mitigation, does not mention silent orphaning of un-FK'd children, does not mention deployment having no cascade param) · **Rec: Keep with edits, fold cascade-broken finding into body** | [issue-002.md](issue-evaluations/issue-002.md) |
| 3 | open | Research: Time field encoding — strict ISO 8601 string requirement vs. numeric timestamp acceptance | complete | Validity: confirmed (rejection) · Legitimacy: spec-conformant rejection + UX defect + sibling silent-failures · Accuracy: accurate symptom / misleading error message · Completeness: incomplete (omits silent drop on `phenomenonTime` and on Datastream `TimeRange`) · **Rec: Close as research-resolved (no leniency change); F1/F2/F3 filed as #18/#19/#20** | [issue-003.md](issue-evaluations/issue-003.md) |
| 4 | open | Research: NaN handling for numeric observation fields — rejection vs. nilValue support | complete | Validity: confirmed (rejection) · Legitimacy: spec idiom (`"NaN"` string) is the publisher convention; rejecting it is non-conformant; **`nilValues` plumbed-but-unconsumed = functional gap** · Accuracy: accurate symptom; mis-framed as research spike · Completeness: incomplete (missed the `null`-vs-omit asymmetry on `optional:true`, missed the open-schema policy for extra fields) · **Rec: Close as research-resolved with implementation owed; F1/F2/F3 filed as #21/#22/#23** | [issue-004.md](issue-evaluations/issue-004.md) |
| 5 | open | Research: Strict schema validation — requiring ALL declared fields in observation results | complete | Validity: confirmed (rejection on missing required field) · Legitimacy: spec-conformant — `optional: true` mechanism is implemented and works · Accuracy: issue mis-frames cs-go as "always strict" when it is actually "required-by-default with per-field opt-out" · Completeness: incomplete (does not mention the `Optional *bool` schema field that already exists, treats SensorHub leniency as the baseline) · **Rec: Close as research-resolved, no code changes, no new follow-ups (defects in this area already covered by #21/#22/#23)** | [issue-005.md](issue-evaluations/issue-005.md) |
| 6 | open | Research: Cross-resource references — `@link` objects only vs. flat `@id` strings for interoperability | complete | Validity: confirmed (cs-go emits `@link`, not `@id`, on Datastream) · Legitimacy: spec-conformant — OGC 23-002 mandates `@link` for cross-collection refs (Datastream/ControlStream/Deployment/System/SamplingFeature) and `@id` only for in-collection siblings (Observation→datastream, Command→controlstream); SensorHub's `system@id`-on-Datastream is the non-conformant variant · Accuracy: issue's comparative table inverts the conformance polarity · Completeness: incomplete (does not enumerate which fields use which encoding per spec; conflates discoverability with interop fix) · **Rec: Close as research-resolved, no code changes against this issue. Two minor polish items filed as #24 (P3 absolute href) and #25 (P4 populate rel/type/title)** | [issue-006.md](issue-evaluations/issue-006.md) |
| 7 | open | `?uid=` query parameter is silently ignored — server returns all resources unfiltered | complete | Validity: confirmed (`?uid=` is silently ignored on HEAD) · Legitimacy: **NOT a spec-compliance bug** — the OGC CSAPI spec does not define a `uid` query parameter; the spec parameter is `id` and its value may be either local IDs OR URIs (canonical source: `opengeospatial/ogcapi-connected-systems/.../parameters/idList.yaml`). cs-go correctly implements the spec parameter (`id IN ? OR unique_identifier IN ?` across all eight UID-bearing repositories) and `?id=urn:...` returns the matching resource live (T2). The "silent ignore" of `?uid=` is identical behavior to any other unknown parameter (verified `?foo=bar` returns the same unfiltered result, T7). · Accuracy: factual claims about cs-go code are accurate; spec citation ("OGC 23-001 §7.3 defines `uid`") is incorrect · Completeness: incomplete — does not mention that the spec parameter is `id` and accepts URIs · **Rec: Close as research-resolved, NOT a P1 bug. Publisher fix is one character (`?uid=` → `?id=`). Filed #26 (P4 docs) only.** | [issue-007.md](issue-evaluations/issue-007.md) |
| 8 | open | `/deployments` only returns top-level deployments — subdeployments not discoverable without parent ID | complete | Validity: confirmed (T1 numberMatched=1, parent only) · Legitimacy: **spec-conformant defect** — canonical Part 1 OAS for `/deployments` does NOT include the `recursive` parameter and does NOT state any top-level-only default (unlike `/systems` which explicitly does). `paths/subdeployments.yaml` adds an explicit spec note that "individual members can also be retrieved by ID directly at the canonical Deployment resources endpoint", which cs-go violates. · Accuracy: main thrust correct; two specific claims wrong — (a) "consistent with how /systems returns all systems" is false (`/systems` is also top-level-only by default per spec; cs-go matches), (b) "`recursive=true` at top-level is a no-op" is false (T2 returns parent + child). · Completeness: incomplete — missed two additional defects under same root cause: D1 (`?parent=X` at top-level returns 0, T3 — logic conflict with `IS NULL`) and D2 (`?id=<sub-URI>` returns 0, T6 — directly violates the spec note above). All three defects collapse into the same one-block fix. · **Rec: Keep open. Apply Option A (corrected) — remove unconditional `parent_deployment_id IS NULL` clause; closes #8 + fixes D1 + fixes D2. P2 reasonable; defensible P1 because D2 violates an explicit spec sentence. No separate follow-ups filed (single fix). Upstream PR candidate.** | [issue-008.md](issue-evaluations/issue-008.md) |
| 9 | open | Default pagination limit of 10 is unusually low — causes silent result truncation for typical workloads | complete | Validity: confirmed (Limit:10 hardcoded; T1 default GET /systems returns features.Count=10 of numberMatched=13) · Legitimacy: **NOT a defect** — canonical CSAPI parameters/limit.yaml literally specifies `default: 10` and (unlike parent OGC API Features) does NOT include the "values are examples and can be changed" escape clause. cs-go matches the spec exactly. · Accuracy: "unusually low" framing contradicted by issue's OWN comparative table (pygeoapi=10, ldproxy=10, cs-go=10 — only SensorHub=100). · Completeness: misses that #7 was already research-resolved, which the issue body says obsoletes the elevated severity ("becomes cosmetic once #7 is fixed"). · **Rec: Close as wontfix/not-a-defect, OR defer to upstream maintainer UX call. cs-go is spec-conformant. Publisher-side fix (pagination handling) belongs in OSHConnect-Python, not cs-go. No follow-ups filed.** | [issue-009.md](issue-evaluations/issue-009.md) |
| 10 | open | SensorML `documents` array silently dropped — system thumbnails/media links lost | not-started | — | — |
| 11 | open | Temporal query parameters (resultTime, phenomenonTime, datetime) silently ignored — all values discarded, including `latest` | not-started | — | — |
| 12 | open | Datastream `unique_identifier` enforces global uniqueness — should be scoped per parent system | not-started | — | — |

`Eval status` values: `not-started` / `in-progress` / `complete`.

---

## Process notes

- **Order.** Evaluations proceed in numeric issue order by default. The user may
  reorder at any time; the tracking table reflects the actual order taken.
- **Branch.** All evaluation work lives on the `issues-evaluation` branch
  (created off `main`). Each evaluation record lands as a separate commit so the
  history of judgments is auditable.
- **No code changes.** This effort produces only documentation. Source code is
  not modified by this plan. Any code-change recommendations that emerge are
  recorded as recommendations only.
- **Plan amendments.** This file may be amended as we learn (e.g. discovering
  that a dimension is too coarse, or that the rubric needs another verdict
  value). Amendments are committed separately from per-issue work and noted in a
  changelog at the bottom of this file.

---

## Changelog

- **2026-04-30** — Plan created. 12 issues open at start (#1–#12). No
  evaluations begun.
- **2026-04-30** — Issue #1 evaluated. Verdict: keep with edits. Surfaced a
  partial-fix-without-closure pattern in commit `1562201` and a compile break in
  `generators_datastream.go`. Cross-referenced #12 as sibling.
- **2026-04-30** — Plan amended: added "Authoritative resources" section
  pointing at the bundled OGC OpenAPIs and pre-digested requirements docs in the
  sibling planning repo (`OS4CSAPI/ogc-client-CSAPI_2:phase-7/docs/research`).
  Replaced flat per-issue template with a per-claim Evidence-block template
  adapted from `phase-6/findings-report-template.md`. Added mandatory
  "Sources Consulted" header section. Issue #1 record updated retroactively to
  match new template and to resolve the OGC 23-002 §9.2 spec question using the
  bundled Part 2 OpenAPI.
- **2026-04-30** — Issue #1 record strengthened: cross-fork verification confirms compile break is in the maintainer's canonical `SomethingCreativeStudios/connected-systems-go` HEAD too (not an OS4CSAPI-fork artefact). Live deployment probe at `https://129-80-248-53.sslip.io/csapi-go/` shows the deployed binary is pre-`1562201` (response includes populated `uid` field that current struct cannot produce). references.md cross-checked: only Part 1 + Part 2 bundled OAS31 are authoritative for CSAPI resource schemas; SensorML/SWE Common JSON Schemas govern other layers.
- **2026-04-30** — Issue #1 record corrected: scope of the build break narrowed. `go build ./cmd/server` succeeds (EXIT=0); only `go build ./...` and `go test ./...` fail because `internal/model/generators/` is a test-fixtures helper that still references the removed `CommonSSN`. Earlier framing of "the project doesn't build" was an overstatement. **Methodology note for future evaluations:** when reporting a build failure, always run the deployable-artefact build separately from the module-wide build before claiming "the project is broken" — they are different signals. Test-only breaks are still defects but should be characterised as such.
- **2026-04-30** — Parallel HEAD deployment brought up at `https://129-80-248-53.sslip.io/csapi-go-head/` (cs-go `4b99421`, fresh DB on port 8283 alongside the existing pre-`1562201` stack on 8282). Project name `csapi-head`, separate Postgres volume `postgres_data_head`, dedicated Caddy route `/csapi-go-head/*`. The Dockerfile builds only `cmd/server/main.go` so the §2.5 test-fixtures break is not a build blocker for the deployed image. This deployment is a clean A/B testbed for the rest of the issue evaluations and answers several open questions on issue #1 directly.
- **2026-04-30** — Issue #1 record extended with empirical live evidence (§2.6 update + new §2.8): three POST variants against `/systems/{id}/datastreams` on HEAD (no-uid, repeat, stray-uid) all return `201 Created` with no `uid` in responses; HEAD-initialized DB has no `unique_identifier` column or unique index; production DB still has both. Open questions §7.2 (live DB schemas) resolved. Three new follow-up surfaces noted (§7.6 stray-uid silent-drop, §7.7 `/api` is a stub, §7.8 capital-S `Systems` field leaks into JSON) — these will be filed as separate issues if they aren't already tracked. Raw transcripts saved under `docs/research/evidence/issue-001/`.
- **2026-04-30** — Issue #2 evaluated. Verdict: keep with edits. Symptom (DELETE → 500 on linked resources, blocks teardown) is fully reproducible on HEAD and prod. Causal model in issue body is partially wrong: (a) blocking FKs are on GORM-generated m2m **join tables** (`fk_system_datastreams_*`, `fk_system_controlstreams_*`), NOT on natural parent→child columns — `observations.datastream_id` and `commands.control_stream_id` have **no FK at all**, so DELETEs of un-linked resources silently orphan their children; (b) response body is sanitized (`{"error":"Failed to delete X"}`, 40B), the raw PG error is in server logs only — refutes issue's claim of raw-error exposure; (c) `?cascade=true` IS exposed on Datastream/ControlStream/System handlers but is **empirically broken** (T8/T11 both 500 — cascade impl deletes children but does not clear the m2m join, then the resource delete trips the same FK); (d) `DeploymentRepository.Delete` has no cascade parameter at all. Evaluation results posted as [comment on #2](https://github.com/OS4CSAPI/connected-systems-go/issues/2#issuecomment-4355934881). Two follow-up issues filed via `mcp_io_github_git_issue_write`: **#16** ([bug] DELETE cascade is broken across resource types) bundles cascade-impl breakage + missing cascade param on Deployment + silent-orphaning hazard into one umbrella for coordinated fix; **#17** ([enhancement] DELETE handlers map every error to 500) is the orthogonal contract fix (23503 → 409, ErrRecordNotFound → 404). **Methodology note:** PowerShell SSH heredoc nested-quoting is a footgun for `psql -c "… contype='f' …"` — escaped single-quotes get stripped silently and yield zero rows that look like "no FKs exist". Use `chr(102)` or shell-escape carefully; cross-check the result against actual server-log error messages before drawing structural conclusions. **Process note:** all evaluation work gets posted upstream — comment on the evaluated issue + new follow-up issues + local docs commit, not just local docs. Raw transcripts saved under `docs/research/evidence/issue-002/`.
- **2026-04-30** — Issue #3 evaluated. Verdict: research-resolved (no leniency change recommended). The original conformance question has a clear spec answer: OGC 23-002 OAS31 defines `phenomenonTime`/`resultTime` as `type: string, format: date-time`, so cs-go's rejection of numeric epoch is correct per spec; SensorHub's leniency is a non-conformant convenience. The *productive* findings came from live A/B testing on HEAD: (a) the rejection error message `"resultTime is required"` is **misleading** — the field IS present, just wrong type — this is the actual UX defect that prompted the spike (filed as **#20**, P3 UX); (b) numeric `phenomenonTime` is **silently dropped** with HTTP 201 — asymmetric vs. `resultTime`'s reject (filed as **#19**, P2 data-integrity); (c) `TimeRange.UnmarshalJSON` (used by Datastream `phenomenonTime`/`resultTime`) **silently swallows** non-string array elements AND silent-drops parse failures, so Datastream POST with numeric or malformed time arrays returns 201 with empty TimeRange (filed as **#18**, P2 data-integrity). Evaluation posted as [comment on #3](https://github.com/OS4CSAPI/connected-systems-go/issues/3#issuecomment-4356023583). Raw transcripts + spec extract under `docs/research/evidence/issue-003/`. **Methodology note:** when a research spike asks "should we accept X?", run live tests for both X-rejected paths AND look for sibling fields where the same wrong-type input leaks through silently — asymmetric strictness across siblings is usually a worse defect than either pure-strict or pure-lenient.
- **2026-04-30** — Issue #4 evaluated. Verdict: research-resolved with implementation owed. The spike's core question ("should we accept `NaN` / `null` / sentinel?") has a clear spec answer — SWE Common's per-field `nilValues: [(reason, value), …]` table IS the canonical mechanism, and the OGC OAS31 examples explicitly use `value: NaN` / `'-Infinity'` / `+Infinity` for Quantity components. The publisher's `"NaN"` string is therefore the spec idiom (RFC 8259 forbids bare `NaN`/`Infinity` literals). **Headline finding:** `DatastreamDataComponent.NilValues` is plumbed through the data model and round-trips on GET, but the result validator (`validateDataComponentValue` in `internal/api/observation_schema_validation.go`) never reads it — `grep -rn NilValues internal/api/` returns zero hits. Same defect ships in the parent fork (sha `2bbb6202`). 16 live test cases on HEAD confirm: T1/T2 — `"NaN"` rejected identically whether nilValues is declared or not; T6 — numeric `-999.0` accepted as a Quantity by type coincidence regardless of nilValues; T11 — extra undeclared fields silently accepted and persisted (open-schema policy, undocumented); T14/T15 — `optional:true` covers omission but NOT explicit `null` (asymmetric absence, same pattern as #19). Evaluation posted as [comment on #4](https://github.com/OS4CSAPI/connected-systems-go/issues/4#issuecomment-4356154026). Three follow-ups filed: **#21** ([P2 bug] result validator ignores `nilValues` table — the headline functional gap), **#22** ([P3 bug] `null` on `optional:true` numeric rejected while omission accepted — sibling of #19), **#23** ([P3 docs] document the result-schema validation contract: required/optional, null semantics, nilValues, open-schema policy for extra fields). Raw transcripts + spec extract + static-analysis snapshot under `docs/research/evidence/issue-004/`. **Methodology note:** "plumbed-but-unconsumed schema fields" is a sibling pattern to "type-assert-and-discard" — fields like `NilValues`, `Constraint`, `Updatable` survive the JSON round-trip but no code reads them in the request path. Future audits should grep `internal/api/` for consumers of every declared schema field, not just inspect the model definitions.
- **2026-04-30** — Issue #5 evaluated. Verdict: research-resolved, no code changes, no new follow-ups. The issue's premise ("server strictly requires ALL declared fields") is **mistaken**: cs-go is required-by-default with per-field `optional: true` opt-out, exactly per SWE Common 3.0 §7.4. `DatastreamDataComponent.Optional *bool` is in the model (line 225) and is consumed correctly in the `datarecord` (line 92) and `vector` (line 115) branches of `validateDataComponentValue`. T1/T2 prove the optional mechanism works end-to-end: identical 6-field schema, same payload omitting `timestamp` — DS_ALLREQ rejects (400), DS_TS_OPT (with `timestamp: optional:true`) accepts (201). The OSHConnect-Python publisher's `duplicate-resultTime-into-result.timestamp` workaround was unnecessary; declaring `timestamp` as optional in the datastream schema would have worked. T6 (typo case) confirms the open-schema sibling defect already filed under #23: a typo'd field name is silently accepted as an extra while the declared field reports missing. Evaluation posted as [comment on #5](https://github.com/OS4CSAPI/connected-systems-go/issues/5#issuecomment-4356212161). Defects in this area already tracked under #21/#22/#23. **Methodology note:** when an issue claims "the server is too strict", first verify whether the spec mechanism for opt-out is implemented and whether the publisher used it. Sometimes the right answer is publisher-side discoverability, not server leniency. Deferred audit: `datachoice`/`dataarray`/`matrix` branches do not consume `optional` on subcomponents; not pursued here because no observed publisher hits this.
- **2026-04-30** -- Issue #6 evaluated. Verdict: research-resolved, no code changes against this issue. **Headline:** the issue's comparative table inverts the conformance polarity. OGC 23-002 mandates `@link` (object) for cross-collection refs (Datastream/ControlStream system/procedure/deployment/featureOfInterest/samplingFeature; Deployment platform/deployedSystems; SamplingFeature sampledFeature; System systemKind; Observation procedure/result; Command procedure) and `@id` (flat string) only for in-collection siblings (Observation.datastream@id REQUIRED, Command.controlstream@id REQUIRED, CommandStatus.command@id REQUIRED, plus same-collection samplingFeature@id on obs/cmd). There is **no `system@id` field defined on Datastream or ControlStream** anywhere in the spec — SensorHub's emission of it is the non-conformant variant. cs-go's model (`internal/model/domains/*.go`) and serialization (`json_formatters/datastream_json.go`) match the spec field-for-field and emit the conformant shape end-to-end (verified live on HEAD: T1/T3 transcripts in `docs/research/evidence/issue-006/`). Recommendation against emitting both `system@id` AND `system@link` on Datastream — would be a non-spec extension creating two sources of truth. Client-side fix is correct (already done in ogc-client-CSAPI_2 #166). Evaluation posted as [comment on #6](https://github.com/OS4CSAPI/connected-systems-go/issues/6#issuecomment-4356255462). Two minor polish items filed: **#24** ([P3 enhancement] inline `@link.href` should be absolute URI to match spec `format: uri` and the existing `links[]` convention), **#25** ([P4 enhancement] populate `rel`/`type`/`title`/`uid` on inline `@link` objects — pure UX win, zero spec risk). **Methodology note:** this was the inverse of #4. For #4, model fields were plumbed but not consumed; for #6, every field IS consumed and IS conformant — the issue's premise was a client expectation calibrated on a different (non-conformant) server, projected onto the conformant one as a 'missing feature'. Future spikes that ask 'why doesn't server X look like server Y?' should start with the spec inventory, not the comparative table; if the spec defines exactly one shape, the asymmetry is a defect on the side that lacks the conformant shape, not on the side that has it.
- **2026-04-30** -- Issue #7 evaluated. Verdict: research-resolved, **NOT** a P1 spec-compliance bug as filed. **Headline:** the OGC CSAPI spec does not define a `uid` query parameter. Direct inspection of the canonical opengeospatial source (`api/part1/openapi/parameters/idList.yaml`) shows the parameter is named `id` with description 'List of resource local IDs or unique IDs (URI)' and explicit examples for both Local IDs and URIs. The same shape is referenced from every list endpoint via `  id`. cs-go's repositories correctly implement the spec dual-match: `query.Where("id IN ? OR unique_identifier IN ?", params.IDs, params.IDs)` across system/datastream/control_stream/deployment/procedure/property/sampling_feature/feature repos. Live verification on HEAD: T2 `GET /systems?id=urn:test:issue1:sys:1` returns numberMatched=1 (the spec URI branch works); T3 `GET /systems?uid=urn:...` returns numberMatched=12 (silently ignored — same as T7 `?foo=bar` which also returns 12, confirming the mechanism is generic 'unknown parameter' fall-through, not a missing implementation). The publisher's reported workaround (fetch `?limit=1000` + client-side scan) is unnecessary — replacing `?uid=urn:...` with `?id=urn:...` works on cs-go today and on any spec-compliant server. The issue's citation 'OGC 23-001 §7.3 defines uid as a standard query parameter' is incorrect — that paragraph defines `id`. Severity downgraded from P1-Critical to non-defect. Evaluation posted as [comment on #7](https://github.com/OS4CSAPI/connected-systems-go/issues/7#issuecomment-4356304779). One follow-up filed: **#26** ([P4 docs] document that `?id=` accepts both local IDs and URIs — the spec UID filter is `?id=`, not `?uid=`); explicitly NOT filing the `uid`→`id` alias as a follow-up because it would be a non-spec extension and the publisher fix is one character. **Methodology note:** this is the third issue in a row (#5 -> #6 -> #7) where comparative-server framing inverted the conformance polarity — SensorHub's behavior was treated as the baseline and cs-go's deviation from SensorHub was read as a defect; in all three cases direct spec inspection showed cs-go matches the spec and SensorHub deviates. The decisive evidence here was a single fetch of `parameters/idList.yaml` from opengeospatial. Rule going forward: when an issue says 'server X is missing feature Y that server Z has', open the canonical OGC source for the feature **by name** before stopping at the comparative table.
- **2026-04-30** -- Issue #8 evaluated. Verdict: **VALIDATED with corrections.** First issue in the run where the spec sided with the report on its main thrust. Spec evidence (canonical Part 1 OAS): /systems explicitly says "By default, only top level systems are included" and includes the 
ecursive parameter; /deployments says "List or search **all** Deployment resources" and does **not** include 
ecursive. paths/subdeployments.yaml adds the explicit guarantee "individual members can also be retrieved by ID directly at the canonical Deployment resources endpoint." cs-go's unconditional WHERE parent_deployment_id IS NULL on /deployments default is a non-spec restriction. Live verification on HEAD (created urn:test:issue8:dep:parent + urn:test:issue8:dep:child via canonical + subdeployments POST endpoints): T1 GET /deployments=1 (parent only), T2 GET /deployments?recursive=true=2 (parent+child) **contradicting the issue's no-op claim**, T3 GET /deployments?parent={pId}=0 **NEW defect D1** (logic conflict — parent filter AND'd with IS NULL → impossible), T4 /{pId}/subdeployments=1 ok, T5 /deployments/{cId} direct=found ok, T6 GET /deployments?id=urn:test:issue8:dep:child=0 **NEW defect D2** (canonical URI lookup of subdeployment fails — directly violates spec note). Two specific claims in the issue body are factually wrong: (a) "/systems returns all systems" — false, system_repository.go:386-389 also defaults top-level only via if !params.Recursive; (b) "
ecursive=true is a no-op" — false, T2 returns 2. Recommendation: keep open, apply Option A (refined with spec citation in comment) — single block change closes #8 + D1 + D2 simultaneously. P2 reasonable; defensible P1 case via D2 (explicit spec sentence violated). No separate follow-ups filed because all three defects share one fix. Strong upstream PR candidate. Methodology rules added: (1) never accept a comparative-server claim without fetching the canonical OAS for both endpoints by name; (2) when a code branch is described as a no-op via comment, run it before believing the comment. Comment posted at #8: https://github.com/OS4CSAPI/connected-systems-go/issues/8#issuecomment-4356426711
- **2026-04-30** -- Issue #9 evaluated. Verdict: **NOT a defect.** cs-go matches the canonical CSAPI spec default (parameters/limit.yaml: default: 10) literally. Unlike the parent OGC API Features spec, CSAPI's limit.yaml does NOT include the "values are examples and can be changed" escape clause — the value is stated literally. Issue's own comparative table refutes its own framing: pygeoapi=10, ldproxy=10, cs-go=10 — only SensorHub=100 deviates. "Unusually low" concentrates entirely on SensorHub-as-baseline (same pattern as #5/#6/#7). Live verification on HEAD: T1 default GET /systems returns features.Count=10 (numberMatched=13); T2 ?limit=5 returns features.Count=5; baseline ?limit=1000 returns features.Count=13. Static claim accurate; spec premise inverted. The issue's own self-downgrade ("becomes cosmetic once #7 is fixed") landed right — #7 was research-resolved, so the publisher's ?limit=1000 workaround is unnecessary because the spec parameter id accepts URIs and one-character ?uid= -> ?id= removes the find_by_uid pagination dependency entirely. Recommendation: close as wontfix/not-a-defect or defer to upstream maintainer UX call. Raising default to 100 would deviate FROM the literal CSAPI spec but TOWARD the parent OGC API Features value — a deployer UX choice, not a spec fix. No follow-up issues filed against cs-go because there is nothing to fix; pagination-handling fix belongs in OSHConnect-Python. Methodology rules confirmed: (1) Even an issue's own comparative table can refute its own framing — cross-check evidence against framing; (2) Self-downgrade clauses are often correct, check dependency-resolution status early; (3) SensorHub-as-baseline pattern now confirmed in 4 of 9 issues evaluated. Comment posted: https://github.com/OS4CSAPI/connected-systems-go/issues/9#issuecomment-4356593770
