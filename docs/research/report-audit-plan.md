# Report Audit Plan — systematic verification of reports 01–13 and the backlog

> **Purpose.** Reports 01–13 in
> [`upstream-issue-reports/`](./upstream-issue-reports/) were drafted
> across many sessions, several of which involved long context, summary
> compaction, and signs of drift in the drafting agent. None of these
> reports has been filed upstream yet. Before any of them are filed,
> each must be mechanically verified for correctness of every citation,
> claim, and recommendation. This plan is the procedure for doing so.

---

## 0. Drift-resistance design principles

These principles drive every choice in §1–§4. Failures of past
sessions inform them.

1. **One report per pass.** No multi-report batches. Every audit
   pass takes exactly one report through the full pipeline before
   the next one starts. This prevents cross-report context bleed.
2. **Mechanical first, subjective last.** Every audit pass starts
   with a fixed checklist of *yes/no* citation checks (does this
   SHA exist, does this line number match, does this file exist).
   Subjective claims (severity, framing, scope) are only reviewed
   after the mechanical layer passes.
3. **Fresh-context subagent for the mechanical layer.** The
   mechanical checklist is dispatched to a search subagent with
   only the report file and the upstream repo path — no session
   memory, no prior conclusions. The subagent reports findings
   verbatim; the orchestrator (the main session) does not summarize
   them away.
4. **Per-report verdict, not per-batch.** Each audit pass ends with
   a discrete verdict (`pass` / `amend` / `retract` / `rewrite`)
   recorded in an audit log. No bulk decisions.
5. **No filing until audit complete.** The "Public-facing extract"
   §10 of each report is the upstream issue body draft. None of
   those go to GitHub until that report's audit verdict is `pass`.
6. **Backlog reconciled last.** The backlog
   ([`upstream-followup-backlog.md`](./upstream-followup-backlog.md))
   may need amending after individual reports are fixed; do it
   once at the end with the corrected reports as input.
7. **Self-review is not enough.** The orchestrator does not
   approve its own reports based on its own re-read. Subagent
   findings or external (user) review must be the gating signal.

---

## 1. Inventory of artifacts to audit

### 1.1 Reports

13 synthesis reports in
[`upstream-issue-reports/`](./upstream-issue-reports/) (note:
`report-05` is intentionally absent — the slot was reassigned during
drafting):

| # | File | Source/parent | Severity (claimed) |
|---|---|---|---|
| 01 | [`report-01-api-endpoint-stub.md`](./upstream-issue-reports/report-01-api-endpoint-stub.md) | _to-verify_ | _to-verify_ |
| 02 | [`report-02-datastream-dangling-unique-identifier-sql.md`](./upstream-issue-reports/report-02-datastream-dangling-unique-identifier-sql.md) | _to-verify_ | _to-verify_ |
| 03 | [`report-03-controlstream-systems-json-leak.md`](./upstream-issue-reports/report-03-controlstream-systems-json-leak.md) | _to-verify_ | _to-verify_ |
| 04 | [`report-04-systemevent-not-in-deletecascade.md`](./upstream-issue-reports/report-04-systemevent-not-in-deletecascade.md) | _to-verify_ | _to-verify_ |
| 06 | [`report-06-totimerange-year-0001-silent-discard.md`](./upstream-issue-reports/report-06-totimerange-year-0001-silent-discard.md) | _to-verify_ | _to-verify_ |
| 07 | [`report-07-samplingfeature-id-silent-drop.md`](./upstream-issue-reports/report-07-samplingfeature-id-silent-drop.md) | _to-verify_ | _to-verify_ |
| 08 | [`report-08-empty-string-resulttime-phenomenontime.md`](./upstream-issue-reports/report-08-empty-string-resulttime-phenomenontime.md) | _to-verify_ | _to-verify_ |
| 09 | [`report-09-constraint-orphaned-in-validator.md`](./upstream-issue-reports/report-09-constraint-orphaned-in-validator.md) | _to-verify_ | _to-verify_ |
| 10 | [`report-10-updatable-orphaned-in-validator.md`](./upstream-issue-reports/report-10-updatable-orphaned-in-validator.md) | _to-verify_ | _to-verify_ |
| 11 | [`report-11-inline-link-absolutization-remaining-5.md`](./upstream-issue-reports/report-11-inline-link-absolutization-remaining-5.md) | issue #24 / `d2d1347` | P3 |
| 12 | [`report-12-inline-link-type-title-uid-enrichment.md`](./upstream-issue-reports/report-12-inline-link-type-title-uid-enrichment.md) | issue #25 / `3fa1b0c`+`704a9e3`+`d2d1347` | P4 |
| 13 | [`report-13-strict-decoder-osh-publisher-bootstrap-regression.md`](./upstream-issue-reports/report-13-strict-decoder-osh-publisher-bootstrap-regression.md) | _to-verify_ | _to-verify_ |

The "_to-verify_" entries are populated as part of the audit pass
for each report. **They are not filled in from session memory.**

### 1.2 Backlog

- [`upstream-followup-backlog.md`](./upstream-followup-backlog.md) —
  the inventory the reports were drafted against. Audited in §3
  after all reports are individually verified.

### 1.3 Out of scope for this audit

- Eval files in `issue-evaluations/`.
- Evidence files in `evidence/`.
- Plan files in `upstream-issues/`.
- These are the *inputs* the reports cite. The reports' job is to
  cite them faithfully; verifying the upstream evals/evidence is
  a separate, larger exercise the user has not asked for.

---

## 2. Per-report audit pipeline

For each report, run §2.1 → §2.2 → §2.3 in order. Do not start the
next report until the current one's verdict is recorded.

### 2.1 Mechanical citation check (subagent)

**Dispatch a search subagent** with the report file path and the
upstream repo path. Subagent prompt template:

> You are auditing a single research report for citation accuracy.
> You have no prior context about this report. Your job is to
> mechanically verify every external claim against the cited source
> of truth. **Do not interpret, summarize, or evaluate the
> recommendation; only check whether the citations are accurate.**
>
> **Report under audit:** `<absolute path to report-NN-*.md>`
> **Upstream repo:** `c:\Users\sbolling\Documents\connected-systems-go`
> **Upstream remote:** `upstream/main` (already fetched)
>
> For each item below, produce one line: `OK` / `FAIL: <reason>` /
> `N/A: <reason>`. Cite the exact line in the report where the
> claim appears.
>
> 1. **Header table** — every URL/path/file referenced exists.
> 2. **Source fork issue numbers** — every `#NN` referenced exists
>    in `OS4CSAPI/connected-systems-go` (use `gh issue view NN -R OS4CSAPI/connected-systems-go`).
> 3. **Commit SHAs** — every SHA is reachable in `upstream/main`
>    (`git cat-file -e <SHA>` or `git log --oneline upstream/main | grep <SHA>`).
> 4. **Commit messages** — when the report quotes a commit subject,
>    it matches `git log -1 <SHA> --format='%s'`.
> 5. **File paths** — every `internal/...` or `src/...` path
>    referenced exists in `upstream/main` (`git cat-file -e
>    upstream/main:<path>`).
> 6. **Line numbers** — when the report cites `<file>:<line>`, the
>    cited line in `git show upstream/main:<file>` actually
>    contains the symbol/expression the report attributes to it.
>    Allow ±2 lines of fuzz; flag larger drifts.
> 7. **Code snippets** — when the report shows a code block as
>    "from `<file>`", the snippet's substantive content
>    (function names, struct fields, key tokens) matches the file
>    in `upstream/main`. Whitespace/comment differences are
>    acceptable; structural differences are not.
> 8. **Cross-references to other reports/plans/evidence** — every
>    relative link resolves to an existing file in the local repo
>    (the workspace path).
> 9. **Spec citations** — only check that the cited spec/document
>    appears in the curated references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>.
>    Do not validate the spec content itself.
> 10. **Internal consistency** — does §1 (re-verification record)
>    match §10 (public extract)? Same SHAs, same file paths, same
>    line numbers, same field names? Flag any divergence.
>
> Return findings as a structured list. Do not propose fixes. Do
> not summarize. Do not editorialize.

**Output:** subagent returns a flat findings list. Orchestrator
copies it verbatim into the audit log (§4) without paraphrasing.

### 2.2 Subjective review (orchestrator + user)

Only run §2.2 if §2.1 returned zero `FAIL` lines. If §2.1 had
failures, jump to §2.3 first to record the verdict, then loop back
to §2.1 after fixes.

The orchestrator reviews these higher-level dimensions and
surfaces them to the user for sign-off:

1. **Severity calibration.** Does the claimed severity (P1/P2/P3/P4)
   match the spec-conformance / data-integrity / UX dimension the
   report's evidence actually demonstrates? Compare against the
   severity rubric implicit in reports 01–10 (which the user has
   already signed off on at filing time).
2. **Framing accuracy.** Does the report's "Claim" §match what
   the static evidence demonstrates? Reports 11 and 12 both
   discovered "plan-vs-reality reframes" during re-verification —
   audit each report for whether its framing is anchored to
   verified static evidence or to outdated plan framing.
3. **Scope guard.** Does the §"Scope guard" actually carve out
   the right adjacent concerns? Cross-check against the carve-outs
   listed in companion reports (e.g., 11 carves out 12's scope and
   vice versa).
4. **Recommended fix realism.** Is the fix mechanically applicable
   against `upstream/main` HEAD as of the audit date? If the fix
   references a helper or pattern that has since been refactored,
   the recommendation needs amending.
5. **Public extract self-containment.** Can the §10 issue body
   stand alone as an upstream issue without the reader needing to
   read the rest of the report? Specifically: does it cite enough
   evidence inline that a maintainer can act on it?
6. **Companion-report cross-references.** When reports cite each
   other (11 ↔ 12, etc.), are the cross-references symmetric and
   accurate?

The user reviews the orchestrator's findings on each axis and
signs off or requests amendments.

### 2.3 Verdict and disposition

End-of-pass deliverable per report — append to §4 audit log:

| Field | Allowed values |
|---|---|
| Report | `report-NN-<slug>` |
| Audit date | ISO-8601 date |
| §2.1 mechanical findings | count of `OK` / `FAIL` / `N/A` |
| §2.2 subjective findings | count of axes with concerns |
| **Verdict** | `pass` / `amend` / `retract` / `rewrite` / `defer` |
| Action items | bullet list, only if verdict is `amend`/`rewrite` |
| Filing-ready? | `yes` only if verdict is `pass` AND user has signed off on §10 |

**Verdict definitions:**

- `pass` — every citation accurate; framing and recommendation
  hold up; ready to file when user is.
- `amend` — minor corrections needed (line-number drift, typo,
  small framing tweak); single round of edits, then re-audit.
- `retract` — premise of the report is unsound (e.g., parent fix
  closed the gap entirely; defect doesn't reproduce); remove from
  filing pipeline. Keep file but mark RETRACTED in header.
- `rewrite` — claims hold but report structure or framing is
  wrong enough that targeted edits won't fix it; re-draft from
  scratch using verified evidence.
- `defer` — audit cannot complete without external input
  (missing reference list entry, ambiguous spec language, etc.);
  surface blocker to user.

---

## 3. Backlog reconciliation

After all 12 reports have a non-`defer` verdict:

1. Open
   [`upstream-followup-backlog.md`](./upstream-followup-backlog.md)
   in full; read every entry.
2. For each backlog entry: verify the entry's "Status" line matches
   reality (filed / pending / superseded / retracted) using the
   reports' final verdicts.
3. Check that every report's claimed backlog item number (in the
   report Header table) maps to an actual backlog entry.
4. Check that the backlog's sequencing notes (e.g., "Sequence after
   item #15") still hold given any retractions/rewrites.
5. Update the backlog in a single commit (`docs(research):
   reconcile backlog with audit verdicts NN–NN`).

---

## 4. Audit log

A new file at
[`./report-audit-log.md`](./report-audit-log.md) (created when the
first audit pass completes) holds the per-report verdict table and
the verbatim subagent findings for each report.

Structure:

```markdown
# Report Audit Log

## report-NN-<slug>
**Audit date:** YYYY-MM-DD
**Verdict:** <pass|amend|retract|rewrite|defer>

### §2.1 Mechanical findings (subagent, verbatim)
<paste>

### §2.2 Subjective findings (orchestrator)
<paste>

### Action items
- [ ] <if any>

### User sign-off
<date or "pending">

---
```

Append, do not edit prior entries.

---

## 5. Audit ordering

Recommended order — **most-likely-to-be-drifted first**, so problems
surface early and the audit process itself can be tuned:

1. **Reports 01 / 02 / 03 / 04 / 06 / 07 / 08** — oldest, longest
   compaction history, drafted under the hardest context conditions.
2. **Reports 09 / 10** — middle vintage; symmetric pair (constraint
   / updatable validator), so audit them together for consistency.
3. **Reports 11 / 12** — most recent, most direct evidence still
   in session memory; quickest passes.
4. **Report 13** — separate work stream (OSH publisher bootstrap
   regression); audit standalone.
5. **Backlog (§3)** — last.

Rationale: starting with the oldest reports lets the §2.1 subagent
template be tuned against the hardest cases first. By the time
reports 11/12 are audited the template is mature.

---

## 6. Time-boxing and stop conditions

- **Per-report pass:** target ≤ one focused session. If a report's
  audit cannot complete in one session, the verdict is `defer` and
  the blocker is logged, not pushed through.
- **Stop conditions:**
  - Three consecutive reports return `retract` — pause the audit
    and reassess the upstream commit landscape; the parent fixes
    may have advanced past what the reports captured.
  - Any single report reveals an evidence-file (`evidence/` or
    `issue-evaluations/`) that itself has factual errors — pause
    and surface to user; do not silently fix the evidence file.
  - Subagent §2.1 returns a citation-style failure that doesn't
    map to one of the 10 checks — pause and update the template
    before continuing.

---

## 7. What this audit will and will not catch

**Will catch:**

- Wrong commit SHAs / line numbers / file paths / function names.
- Quoted code that doesn't match the upstream file.
- Reports whose premise was closed by a later upstream commit.
- Cross-reference asymmetry between companion reports.
- Severity miscalibration (subjective review).
- Recommendations that no longer apply mechanically against current
  HEAD.

**Will not catch:**

- Spec misinterpretations where the spec text itself is correctly
  quoted but the report's reading of it is wrong. This requires a
  separate spec-reading pass, out of scope.
- Errors in the source eval/evidence files. Those are inputs; the
  reports' job is to cite them faithfully, not to re-verify them.
  If §2.1 surfaces what looks like an evidence-file error, the
  audit logs it and surfaces to user for a separate exercise.
- Subjective judgment calls about which alternative (Option A vs B
  vs C in §5 of each report) is "best." The audit checks that the
  recommendation is internally consistent with the evidence cited;
  it does not litigate the design choice.

---

## 8. Kickoff checklist

Before starting report-01's audit pass:

- [ ] Confirm `upstream/main` HEAD SHA and record it in §4 audit
      log header (so all audit passes share a common reference
      point).
- [ ] Confirm `gh` CLI is authenticated against
      `OS4CSAPI/connected-systems-go` for issue-existence checks.
- [ ] Create the audit log file [`./report-audit-log.md`](./report-audit-log.md)
      with empty header sections.
- [x] **Filing cadence: file each `pass`-verdict report immediately**
      (user decision, 2026-05-05). Trade-off: faster maintainer
      feedback loop; if a later report retracts and breaks a
      cross-reference into an already-filed issue, fix with a
      follow-up comment on the filed issue rather than holding
      the whole queue.
