# Upstream Follow-up Plan

Companion to [`upstream-followup-backlog.md`](./upstream-followup-backlog.md).
Defines the action sequence for settling outstanding fork-side issues and
filing the residual backlog upstream against
`SomethingCreativeStudios/connected-systems-go`.

State as of 2026-05-05: 24 of 26 fork issues closed; 2 awaiting maintainer
response (#10 status, #22 pushback); 16-item backlog ready for upstream
filing. Phase 1 (await maintainer) and Phases 2-5 (file backlog) run in
parallel — we are **not** holding backlog filing on #10/#22 settling.

---

## Phase 1 — Settle outstanding fork issues (await maintainer)

### Action 1 — issue #10 (status, awaiting scope reply)

Status comment 4379050259 asks `@SomethingCreativeStudios` whether the
remaining 7 `documentation→documents` sites (including the input-side struct
`internal/model/domains/system.go:41`) should be in scope.

| Maintainer reply | Our action |
|---|---|
| "Fix the rest" | Keep #10 open. Prepare fork-side PR (or upstream PR if requested). |
| "Out of scope" | Close #10 not-planned. File as fresh upstream issue (new backlog item). |
| No response after reasonable interval | Ping once, then close not-planned and file upstream. |

### Action 2 — issue #22 (pushback, awaiting response)

Pushback comment 4379878786 argues against the null-as-absence change
shipped in `1b2b614` on spec-conformance grounds (no `null` type-union in
OAS31; competes with `nilValues` from #21; weakens schema enforcement).
Asked for either a config flag (off-by-default) or improved diagnostic
messages.

| Maintainer reply | Our action |
|---|---|
| Accepts pushback | Wait for follow-up commit, then close as completed. |
| Rejects pushback | Close not-planned with respectful concession. File the residual concerns (config-flag proposal, diagnostic improvement) as a fresh upstream issue. |
| No response after reasonable interval | Ping once, then close not-planned and file upstream. |

---

## Phase 2 — Triage and prioritize backlog for upstream filing

Re-read each backlog item's source eval before filing. Group filings by
category and severity.

### Tier A — file first (high-confidence P2 defects)

| # | Backlog item | Severity | One-line description |
|---|---|---|---|
| 1 | #14 (backlog) | P2 | `/api` stub — binding spec `SHALL` violation. Recommend Option 3 → Option 1. |
| 2 | #5 (backlog)  | P2 | ControlStream `Systems` JSON leak (sibling of `2dc09f7`). |
| 3 | #12 (backlog) | P2 | `DatastreamDataComponent.Constraint` orphaned in validator (same shape as `nilValues` from closed #21). |
| 4 | #7 (backlog)  | P2 | `SystemEvent` not in `SystemRepository.deleteCascade` — orphaning. |

### Tier B — file in second batch (P3 defects, narrower exposure)

| # | Backlog item | Severity | One-line description |
|---|---|---|---|
| 5  | #4 (backlog)  | P3 | Datastream `applyFilters` dangling `unique_identifier` SQL. |
| 6  | #8 (backlog)  | P3 | Malformed UUID path-param → 400 (currently 500); all path-param UUID handlers. |
| 7  | #9 (backlog)  | P3 | Legacy `ToTimeRange` year-0001 silent-discard at three sites. |
| 8  | #10 (backlog) | P3 | `samplingFeature@id` silent-drop type assertion. |
| 9  | #11 (backlog) | P3 | `resultTime`/`phenomenonTime` empty-string conflation (T5 of decoder matrix). |
| 10 | #13 (backlog) | P3 | `DatastreamDataComponent.Updatable` orphaned in validator. |

### Tier C — file as enhancement batch (all in scope)

| # | Backlog item | Severity | One-line description |
|---|---|---|---|
| 11 | #15 (backlog) | P3 enh | Inline `@link` absolutization for 5 remaining resource types. |
| 12 | #16 (backlog) | P4 enh | Inline `@link` Type/Title/UID enrichment (per maintainer's *"not all fully enriched"* self-ack). |
| 13 | #6 (backlog)  | enh    | Latent untagged relationship slices on System/Procedure. |
| 14 | #14 (backlog list) — Approach 3a | enh | Schema-side FK tags. **Only if maintainer signals interest** after Tier A item #4 (SystemEvent cascade) lands. |

---

## Phase 3 — Bundle items before drafting

Decide which backlog items combine into a single upstream issue **before**
drafting begins. Bundling drops upstream issue count from 14 → ~10-11 and
saves the maintainer review time. This is a sorting step, not a writing
step — it must be settled before Phase 4 starts on any item.

| Bundle | Items | Rationale |
|---|---|---|
| **Decoder hygiene** | Backlog #9 + #11 | Both touch `time_range.go` / observation decode. |
| **Handler UUID & decode** | Backlog #6 + #8 | Both relate to handler-entry hygiene and decode safety. |
| **Inline `@link` completion** | Backlog #15 + #16 | Same JSON formatter sites; same maintainer touch zone. |
| **Validator orphans** | Backlog #12 + #13 | Both are "model declares it, GET round-trips it, validator ignores it" with `Constraint` and `Updatable` on the same `DatastreamDataComponent` struct. |

All other items file as standalone issues.

---

## Phase 4 — Draft and file each item one at a time

**Standing decision (2026-05-05):** All upstream submissions are filed as
**issues**, never as direct pull requests — even for one-line fixes. The
maintainer owns the canonical repo and decides what merges; our job is to
surface findings with full evidence and let them choose the fix shape and
timing. Direct PRs would skip that triage step and risk burning maintainer
trust on unsolicited changes.

**Standing decision (2026-05-05):** Issues are drafted and filed **one at
a time**, not in batches. Same careful pace we used during the closure
pass. No drafting begins on issue N+1 until issue N is filed and the
backlog updated.

**Standing decision (2026-05-05):** Each filing begins with a dedicated
**per-issue research plan** committed to the fork **before** any drafting
work starts. This is a sources-and-organization document, not a
pre-written issue.

**Standing decision (2026-05-05):** Research output is captured in a
**per-issue report** (synthesis document) committed to a separate
subfolder *before* any upstream issue body is drafted. The upstream
issue links to the report; the report is the durable internal record.
This separates internal reasoning (alternatives weighed, fork-side
context, maintainer-triage interpretation) from the public-facing,
maintainer-ready issue text. If an issue is later rejected or
re-scoped, the report still stands as the durable artifact.

Per-item layout:

```
docs/research/
  upstream-issues/
    plan-NN-<slug>.md        <- Step 4.1 research plan (sources to review)
  upstream-issue-reports/
    report-NN-<slug>.md      <- Step 4.2 synthesis (findings, fix recommendation)
```

Workflow per item:

### Step 4.1 — Author the research plan

Create `docs/research/upstream-issues/plan-NN-<slug>.md` (where `NN` is the
sequence number 01..N+ across the filing pass and `<slug>` is a kebab-case
short title). The plan **identifies and organizes all sources of
information** that will be reviewed during issue drafting. It does **not**
attempt to write the issue itself.

**Mandatory first action when authoring or working from any research
plan: re-read the curated authoritative-references list at**
<https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>.

That file is the canonical inventory of approved spec/standard sources
(OGC, OAS, RFC, etc.) for this work stream. The agent has a known failure
mode of "forgetting" which references are authoritative and substituting
self-located ones mid-task, which corrupts citations. The research plan
must (a) include a top-of-file directive to re-open the references list
before each work session on the plan, and (b) record the specific
entries from that list that this issue will cite, so any later session
can re-ground without re-deriving the source set.

The plan must enumerate:

1. **Backlog entry** being addressed (item number + summary line copied
   verbatim from `upstream-followup-backlog.md`).
2. **Source fork issue(s)** — original `OS4CSAPI/connected-systems-go`
   issue number(s) whose closure surfaced this finding. List every one
   for bundles.
3. **Existing evaluation artifacts** — paths to the relevant
   `docs/research/issue-evaluations/issue-NNN.md` and
   `docs/research/evidence/issue-NNN/*` files, with a one-line note on
   what each provides.
4. **Maintainer triage signal** — verbatim quote (if any) from the
   maintainer on the source fork issue, with a note on how it shapes the
   filing.
5. **Upstream commits relevant to the area** — SHAs and one-line
   descriptions of any upstream commits that touched adjacent code, so
   we can cite them in the issue body for context (e.g. *"sibling of
   `2dc09f7`"*).
6. **Re-verification commands** — the exact `git`/`grep`/`curl`
   commands the drafter must run on `upstream/main` HEAD before
   writing, to confirm the defect is still live.
7. **Spec-authority sources** — URLs / file paths for any OAS, OGC, or
   RFC documents we'll cite. **Every entry here must trace to the
   authoritative references list at
   `OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.**
   If a citation is needed that is not in that list, stop and surface
   the gap to the user before drafting — do not self-source.
8. **Open questions** — anything that needs clarification before
   drafting can finalize.
9. **Drafting checklist** — the sub-bullets of the issue template
   (Context / Claim / Static evidence / Live evidence / Spec authority /
   Recommended fix / Scope guard) with a one-line note per bullet on
   what content will land there. This is the bridge from research plan
   to draft.

### Step 4.2 — Conduct research and produce the report

Once the research plan is committed and the user confirms it captures
everything needed, work through the source inventory in the plan and
produce a synthesis report at
`docs/research/upstream-issue-reports/report-NN-<slug>.md` (matching the
plan's `NN` and `<slug>` exactly).

The report is **internal-facing** — durable record of what was found,
what was ruled out, what alternatives were considered, what the
recommended fix is, and why. It is the document the upstream issue
links to. It is also the document a future session re-grounds from if
context is lost.

**Mandatory first action when authoring the report: re-read the
authoritative-references list** (same URL as Step 4.1) and the research
plan itself. Do not begin synthesis from memory of the plan's contents.

The report must contain, at minimum:

1. **Header block** — backlog item ID, source fork issue refs, plan
   file path, date authored.
2. **Re-verification record** — exact commands run, their output (or a
   trimmed quote of the relevant lines), and the `upstream/main` SHA
   at the time of verification. This proves the defect was still live
   when the report was written.
3. **Static evidence** — file:line citations and the relevant code
   excerpt(s).
4. **Live evidence** — request/response excerpt(s) if applicable.
5. **Spec authority** — verbatim relevant clauses with citation
   traceable to the references list. Flag any precision adjustments
   (e.g. "binding `SHALL` from `landing-page` only; cs-go does not
   declare `/conf/oas30`").
6. **Alternatives considered** — fix options weighed, with pros/cons
   and the reasoning behind the ranking. Internal-only content;
   maintainer doesn't need to see ruled-out paths but we need to
   preserve the reasoning.
7. **Recommended fix** — the option we will lead with in the upstream
   issue, with a one-line rationale.
8. **Scope guard** — explicit "what NOT to touch" list.
9. **Fork-side context** — any relevant fork-only state (workarounds
   shipped, related fork issues, maintainer triage interpretation).
   Internal-only.
10. **Open questions resolved** — bullet list of every open question
    from the research plan, each with its resolution.
11. **Public-facing extract** — a clearly-marked section containing
    *only* the content that should land in the upstream issue body, in
    the issue-template format (Context / Claim / Static evidence /
    Live evidence / Spec authority / Recommended fix / Scope guard).
    This is what Step 4.3 mechanically copies into GitHub.

### Step 4.3 — File the issue

Once drafted and reviewed:

1. **Re-verify on `upstream/main` HEAD** — defect may have been fixed
   in passing. Update the report's re-verification section if the SHA
   has advanced since the report was authored.
2. **Post the issue** to `SomethingCreativeStudios/connected-systems-go`
   using the **public-facing extract** from the report (Step 4.2 §11)
   verbatim. The issue body is a mechanical copy, not a fresh write.
3. **Reference research artifacts** — link to:
   - `docs/research/upstream-issue-reports/report-NN-<slug>.md`
     (primary — the maintainer-facing synthesis)
   - `docs/research/issue-evaluations/issue-NNN.md` (if useful)
   - `docs/research/evidence/issue-NNN/*` (if useful)
4. **Cross-reference** the original `OS4CSAPI/connected-systems-go`
   issue number(s).

### Step 4.4 — Update tracking

After filing:

1. Move the backlog item from "Open items" → "Closed / superseded" with
   the new upstream issue number.
2. Append a one-line entry at the bottom of the report (and the plan)
   recording the filed upstream issue URL and date, so both files
   become the durable record of "what we filed and why" for that item.

---

## Phase 5 — Closure

After all tiers (A + B + C) filed:

1. **Update backlog "Process notes"** with cross-references to filed
   upstream issue numbers.
2. **Final review** — confirm every fork-side closed issue references
   either:
   - a fix (commit SHA), or
   - a not-planned rationale, or
   - a backlog → upstream issue pointer.
3. **Maintainer engagement check** — if maintainer signals interest in
   deeper hardening on Tier A item #4 (SystemEvent cascade), file Tier C
   item 14 (Approach 3a schema-side FK tags). Otherwise leave deferred.

---

## Open decisions

_(none currently)_

## Settled decisions

- **2026-05-05** — All upstream submissions are filed as issues, never
  direct PRs. Rationale recorded in Phase 4.
- **2026-05-05** — Issues are drafted and filed one at a time, not in
  batches. Recorded in Phase 4.
- **2026-05-05** — Tier C is fully in scope. All four enhancements file
  alongside Tier A + B (item 14 / Approach 3a remains contingent on
  maintainer engagement signal per Phase 5).
- **2026-05-05** — Backlog filing does not wait on #10 or #22 settling.
  Phase 1 and Phases 2-5 run in parallel.
- **2026-05-05** — Each upstream filing begins with a dedicated per-issue
  research plan at `docs/research/upstream-issues/plan-NN-<slug>.md`.
  The plan identifies sources only; drafting is a separate step.
  Recorded in Phase 4 (Step 4.1).
- **2026-05-05** — Each filing produces a per-issue **report** at
  `docs/research/upstream-issue-reports/report-NN-<slug>.md` before any
  upstream issue body is drafted. The upstream issue is a mechanical
  copy of the report's "Public-facing extract" section. The report is
  the durable internal record (alternatives weighed, fork-side context,
  re-verification log). Recorded in Phase 4 (Step 4.2).
- **2026-05-05** — Both research plans and reports must begin work
  sessions by re-reading the authoritative-references list at
  `OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.
  Self-sourcing references is forbidden; gaps must be surfaced to the
  user before drafting.

---

## Cross-references

- Backlog: [`upstream-followup-backlog.md`](./upstream-followup-backlog.md)
- Issue evaluations: [`issue-evaluations/`](./issue-evaluations/)
- Evidence files: [`evidence/`](./evidence/)
- Maintainer triage source: tracked in conversation logs;
  reproduced inline in each closure comment for traceability.
