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

Each per-issue file contains:

```markdown
# Issue #NNN — <title>

- **URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/NNN
- **Labels:** ...
- **Evaluated against cs-go commit:** <SHA>
- **Evaluator:** <session>
- **Date:** YYYY-MM-DD

## 1. Issue body (verbatim)
...

## 2. Load-bearing claims
1. ...
2. ...

## 3. Verification
### 3.1 Code claims
...
### 3.2 Spec claims
...
### 3.3 Reproduction
...

## 4. Verdicts
- Validity: ...
- Legitimacy: ...
- Accuracy: ...
- Completeness: ...

## 5. Reasoning
...

## 6. Recommendation
...

## 7. Open questions / unknowns
...
```

---

## Issue inventory & tracking

Snapshot taken at start of effort. State and counts will drift; the per-issue
records are the source of truth as evaluations land.

| # | State | Title (verbatim) | Eval status | Verdict | Record |
|---|---|---|---|---|---|
| 1 | open | Datastream creation without explicit `uid` stores empty string, violates unique constraint on second create | not-started | — | — |
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
