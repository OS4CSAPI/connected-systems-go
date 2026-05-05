# Research Plan 06 — Legacy `ToTimeRange` silent-discard year-0001 pattern

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
item **9**, copied verbatim:

> ### 9. Legacy `ToTimeRange` silent-discard year-0001 pattern
> - **Source:** Issue #18 closure (adjacent finding).
> - **Category:** Defect (P3 — narrow residual exposure).
> - **Summary:** `internal/model/common_shared/time_range.go:159, 164, 173` still
>   use `t, _ := time.Parse(...); startTime = &t` — error discarded AND `&t`
>   taken regardless, producing a pointer to year-0001 instead of nil.
>   Reachable from (a) `UnmarshalJSON`'s string-form branch at
>   `time_range.go:119` (`*tr = ToTimeRange(s)`), so a JSON body of
>   `"phenomenonTime":"junk/junk"` still silently corrupts; and
>   (b) `history.go:39`. Two-part fix: inline-guard the three sites OR
>   deprecate `ToTimeRange` in favor of `toTimeRangeStrict` and migrate the
>   two callers.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P3 — narrow residual exposure**.
> The eval's adjacent finding (issue-018 §"Adjacent finding") frames
> this as **strictly worse** than the parent #18 defect (a pointer to
> year-0001 is more dangerous than `nil` because downstream code may
> not test for `time.IsZero()`). However, blast radius is narrower
> than #18: only the string-form `UnmarshalJSON` branch and
> `history.go:39`. Severity stays **P3** because the parent fix
> already handles the array and object branches. If §6 finds the
> string-form branch is the *most-used* path, escalate to P2 in the
> report — flag for drafter review.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#18` — closed by upstream commit(s)
  applying the strict-parse pattern to `UnmarshalJSON`'s array and
  object branches.
- This filing is the **`ToTimeRange` legacy-pattern residual**
  identified in the eval as the adjacent finding: the helper
  function the string-form branch delegates to was not strict-parsed
  at the same time, so the silent-discard channel survives via that
  path plus `history.go:39`.
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-018.md`](../issue-evaluations/issue-018.md) | Full evaluation. "Adjacent finding (not in original issue body)" §documents the year-0001 pattern at lines 119, 124 (eval-time line numbers; backlog cites 159, 164, 173 — drift expected post-fix; §6 must re-locate). "Suggested fix" §lists both fix shapes (inline-guard the three sites, OR promote `ToTimeRange` to `(TimeRange, error)` and migrate callers). |
| [`../evidence/issue-018/static-analysis-2026-04-30.md`](../evidence/issue-018/static-analysis-2026-04-30.md) | Source for the `t, _ := time.Parse(...)` pattern citations. Documents six silent-discard sites in `UnmarshalJSON` (the parent #18 fix surface) plus two more in `ToTimeRange` (this filing's surface). |
| [`../evidence/issue-018/live-test-2026-04-30.md`](../evidence/issue-018/live-test-2026-04-30.md) | Pre-#18-fix reproduction. **Post-#18-fix live reproducer for the string-form branch must be captured during Step 4.2** — a JSON body of `"phenomenonTime":"junk/junk"` is the canonical trigger per backlog. Expected behavior: 201 Created with the row written and `phenomenonTime` columns set to year-0001 (`0001-01-01T00:00:00Z`). |
| [`../evidence/issue-018/spec-authority-2026-04-30.md`](../evidence/issue-018/spec-authority-2026-04-30.md) | CSAPI Part 1 + RFC 7493 §3.4 require rejection, not silent coercion. Same spec posture as parent #18 — strict-parse is the binding rule. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted issue #18 (a P2
data-integrity defect) and shipped strict-parse for the array and
object branches. Implications:

- Tone: **completing the strict-parse coverage** for the third
  decode path (string-form via `ToTimeRange`) and the secondary
  caller (`history.go`). Frame as "applying the same strict-parse
  pattern you used in `<#18-fix-commit>` to the helper function the
  string-form branch delegates to."
- The maintainer's choice of *which* branches to fix is what makes
  the fix shape a judgment call. Plan must respect both options
  in the recommendation.

## 5. Upstream commits relevant to the area

- The commit(s) that closed `#18` — drafter must identify exactly
  via §6. Cite as the precedent for the fix pattern (`fmt.Errorf` /
  `&decodeError` on parse failure) and confirm exactly which
  branches got the treatment.
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) `UnmarshalJSON` array and object branches are strict-parsed;
  (b) `ToTimeRange` still uses the legacy `t, _ := time.Parse(...);
      &t` pattern;
  (c) `history.go:39` (or current line) still calls `ToTimeRange`.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Read the current time_range.go and locate the legacy pattern:
git show upstream/main:internal/model/common_shared/time_range.go

# Confirm the legacy `t, _ := time.Parse(...)` pattern still present:
git show upstream/main:internal/model/common_shared/time_range.go | Select-String -Pattern 't,\s*_\s*:=\s*time\.Parse|, _ := time\.Parse' -Context 2,2

# Confirm `UnmarshalJSON` array+object branches now strict-parse (sanity, parent fix):
git show upstream/main:internal/model/common_shared/time_range.go | Select-String -Pattern 'UnmarshalJSON|fmt\.Errorf' -Context 3,3

# Find all callers of ToTimeRange:
git grep -nE 'ToTimeRange\(' upstream/main -- '*.go'

# Confirm history.go still calls ToTimeRange:
git grep -nE 'ToTimeRange' upstream/main -- internal/model/common_shared/history.go
git show upstream/main:internal/model/common_shared/history.go | Select-String -Pattern 'ToTimeRange' -Context 2,2

# Recent history on time_range.go (find the closing commit):
git log --oneline upstream/main -- internal/model/common_shared/time_range.go | Select-Object -First 10
```

```pwsh
# Live re-verification — auth required for POST. Sketch:
# POST a Datastream with body: { ..., "phenomenonTime": "junk/junk" }
# Expected: 201 Created, then GET the new datastream and observe
# phenomenon_time_start = 0001-01-01T00:00:00Z (or omitempty-suppressed
# year-0001 in the JSON response, depending on serialization).
# If POST returns 400 instead of 201, the fix has already landed in
# ToTimeRange — stop drafting.
```

Expected on unchanged post-#18-fix state:
- `time_range.go` `UnmarshalJSON` array and object branches return
  `fmt.Errorf` / `&decodeError` on parse failure (parent fix in place).
- `time_range.go` `ToTimeRange` still uses
  `t, _ := time.Parse(...); startTime = &t` at the lines backlog cites
  (159, 164, 173 — verify current line numbers in §6).
- `time_range.go` `UnmarshalJSON` string-form branch still does
  `*tr = ToTimeRange(s)` (delegation site that exposes the silent-discard
  channel).
- `history.go` still calls `ToTimeRange` (second caller).
- Live: POST with `"phenomenonTime":"junk/junk"` → 201 Created, row
  written with year-0001 columns.

If `ToTimeRange` already strict-parses on `upstream/main`, **stop
drafting** — defect already fixed.

If live POST returns 400, parent-fix may have been extended; verify
via `git log` and stop if so.

## 7. Spec-authority sources

This finding has the same spec posture as parent #18: strict-parse
is the binding rule. Direct binding via RFC 7493 + CSAPI Part 1.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **RFC 7493** §3.4 (I-JSON time format) | "RFC 7493 — The I-JSON Message Format" under *IETF RFCs* | Direct binding: I-JSON time values must conform to RFC 3339. Silent acceptance of non-RFC-3339 input violates this. **Primary citation, same as #18.** |
| **OGC 23-001** — CSAPI Part 1 §"Time encoding" / `phenomenonTime` schema | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms `phenomenonTime` is RFC 3339, not free-form. |
| **OGC 23-002** — CSAPI Part 2 Datastream / Observation schemas | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms TimeRange schema and the resources that embed it. |
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: malformed time string is malformed request syntax → 400, not silent acceptance with corrupted state. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — not the binding standard for time-format validation.
- JSON Schema 2020-12 — relevant only insofar as RFC 3339 is the
  format; covered by RFC 7493 citation.

## 8. Open questions

1. **Recommend inline-guard (option A) or strict-rewrite of
   `ToTimeRange` (option B)?**
   - Option A (inline-guard the three sites): smaller diff, no API
     change, pointer remains nil on parse failure. Matches parent
     #18's branch-by-branch approach.
   - Option B (promote `ToTimeRange` to `(TimeRange, error)` and
     migrate the two callers): larger diff but eliminates the
     silent-discard channel structurally. Matches the eval's
     "Suggested fix" §2 framing as a "wider change".
   **Tentative answer: lead with option A** as the smaller-diff,
   maintainer-friendly fix that mirrors parent-commit shape.
   Present option B as the structural alternative for if the
   maintainer prefers a one-time fix that prevents future
   regressions.
2. **Severity P3 vs. P2?** Backlog says P3; parent was P2. Plan
   tentatively retains P3 because the parent fix already covers
   the high-traffic decode paths (array + object). However if §6
   confirms the string-form path (`"phenomenonTime":"junk/junk"`)
   is the canonical client shape (e.g. ISO 8601 interval syntax
   `"start/end"` is the documented user-facing form), this is P2.
   **Tentative answer: drafter must check the documented input
   shape for `phenomenonTime` and `resultTime` in CSAPI Part 1;
   if the string-form is documented as primary, escalate to P2.**
3. **Bundle the `Observation.ResultTime` asymmetric-strictness
   note from issue-018 eval?** No — that's already-strict
   (eval confirms the Observation handler returns 400 on parse
   failure), and citing it here is for *justification* (proves
   silent-discard is a defect not a design choice), not as a
   bundled fix.
   **Tentative answer: cite as a one-paragraph "this is why
   silent-discard is a defect, not a deliberate API choice"
   note in the report, not as a fix-scope item.**
4. **Live reproducer required?** Auth needed for POST. Static
   evidence + clear delegation-site argument should be sufficient.
   **Tentative answer: capture live opportunistically; static-only
   is acceptable.**
5. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–05.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Three sentences: `<#18-fix-commit>` applied strict-parse to `UnmarshalJSON`'s array and object branches. The string-form branch still delegates to `ToTimeRange`, which retains the legacy `t, _ := time.Parse(...); startTime = &t` pattern at three sites. The result is a pointer to year-0001 (worse than `nil`) on malformed input, plus a second exposure via `history.go`. |
| **Claim** | One bullet: `ToTimeRange` produces silent year-0001 pointers for malformed input, reachable via the `UnmarshalJSON` string-form branch and via `history.go`'s direct call. |
| **Static evidence** | Excerpt of `time_range.go` showing the three `t, _ := time.Parse(...); startTime = &t` sites; excerpt of the `UnmarshalJSON` string-form branch showing the delegation; one-line `git grep` showing the two call-sites of `ToTimeRange`. From [`evidence/issue-018/static-analysis-2026-04-30.md`](../evidence/issue-018/static-analysis-2026-04-30.md), refreshed in §6. |
| **Live evidence** | If auth permits: POST with `"phenomenonTime":"junk/junk"` → 201 Created → GET shows year-0001 columns. If not: cite the static delegation chain plus parent #18's live evidence as transitive precedent. |
| **Spec authority** | One paragraph: RFC 7493 §3.4 is the binding rule (RFC 3339 for time). Same spec posture as parent #18; strict-parse is the documented requirement, silent-coerce is the defect. |
| **Recommended fix** | Lead with option A (inline-guard the three sites with `if err != nil { return TimeRange{} }` or equivalent) as the maintainer-friendly small-diff fix. Present option B (promote to `(TimeRange, error)` and migrate `UnmarshalJSON` string-form + `history.go`) as the structural alternative. |
| **Scope guard** | "What NOT to touch": no change to the array/object branches (parent fix in place); no `Observation.ResultTime` refactor (already strict; eval cites for justification only); no broader time-handling changes outside `time_range.go` and `history.go`. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-06-totimerange-year-0001-silent-discard.md`](../upstream-issue-reports/report-06-totimerange-year-0001-silent-discard.md)

Same `NN` and `<slug>`
(`06-totimerange-year-0001-silent-discard`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
