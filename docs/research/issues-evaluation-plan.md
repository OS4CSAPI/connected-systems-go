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
| 2 | open | DELETE on parent resources fails with raw PostgreSQL FK constraint error instead of cascading or returning structured error | not-started | — | — |
| 3 | open | Research: Time field encoding — strict ISO 8601 string requirement vs. numeric timestamp acceptance | not-started | — | — |
| 4 | open | Research: NaN handling for numeric observation fields — rejection vs. nilValue support | not-started | — | — |
| 5 | open | Research: Strict schema validation — requiring ALL declared fields in observation results | not-started | — | — |
| 6 | open | Research: Cross-resource references — `@link` objects only vs. flat `@id` strings for interoperability | not-started | — | — |
| 7 | open | `?uid=` query parameter is silently ignored — server returns all resources unfiltered | not-started | — | — |
| 8 | open | `/deployments` only returns top-level deployments — subdeployments not discoverable without parent ID | not-started | — | — |
| 9 | open | Default pagination limit of 10 is unusually low — causes silent result truncation for typical workloads | not-started | — | — |
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
