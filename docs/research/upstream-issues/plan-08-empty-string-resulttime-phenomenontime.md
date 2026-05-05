# Research Plan 08 — `resultTime` / `phenomenonTime` empty-string conflates with missing

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
item **11**, copied verbatim:

> ### 11. `resultTime` / `phenomenonTime` empty-string conflates with missing
> - **Source:** Issue #20 closure (T5 of original 7-row matrix).
> - **Category:** Defect (P3 — UX residual).
> - **Summary:** `resultTime: ""` passes `(string); ok` but skips the inner
>   `if rtStr != ""` block, falls through to the late `IsZero()` guard, and
>   produces `"resultTime is required"` rather than a distinct empty-string
>   message. Same applies symmetrically to `phenomenonTime`. Trivial fix:
>   add `if rtStr == "" { return nil, &decodeError{msg: "resultTime must be
>   a non-empty ISO 8601 string"} }` between the `ok` check and the inner
>   parse block. The post-`1b2b614` decoder reduced conflation from 6→1 to
>   2→1; this closes the last gap.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P3 — UX residual**. Eval (issue-020)
> classified parent #20 as P3 — UX. Both fall in the same UX/error-message
> category; the residual is mechanically smaller. P3 stands.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#20` — closed by upstream commit
  `1b2b614` (same commit that closed #19; the fixes folded together
  per eval recommendation).
- This filing is the **empty-string T5 residual** identified by the
  backlog: the post-`1b2b614` decoder added explicit branches for
  wrong-type, missing, and null cases, but did not branch on the
  `string-but-empty` case, which still falls through to the late
  `IsZero()` guard and emits the misleading `"resultTime is required"`
  message.
- No other fork issues bundle in. (Note: this is symmetrically
  related to plan-07 in that both are residuals of the `1b2b614`
  fix family, but they touch different decode blocks and should be
  filed separately.)

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-020.md`](../issue-evaluations/issue-020.md) | Full evaluation. The 7-row test matrix (T1–T7) is the source of the "6 distinct client errors → 1 message" finding. The body's proposed fix code (reproduced in the eval's "Suggested fix" §) explicitly includes a separate empty-string branch with the message `"resultTime must be a non-empty RFC 3339 date-time string"` — i.e., this branch is *exactly the residual* that backlog item 11 identifies as missing post-fix. |
| [`../evidence/issue-020/static-analysis-2026-04-30.md`](../evidence/issue-020/static-analysis-2026-04-30.md) | Source for the conjunctive guard pattern citation and the late `IsZero()` rescue mechanism. |
| [`../evidence/issue-020/live-test-2026-04-30.md`](../evidence/issue-020/live-test-2026-04-30.md) | Pre-`1b2b614` 7-row matrix. T5 (empty string) returned `"resultTime is required"`. **Post-`1b2b614` matrix must be re-captured during Step 4.2** to confirm the backlog claim that T5 still returns the misleading message (i.e., the empty-string branch is still missing). |
| [`../evidence/issue-020/spec-authority-2026-04-30.md`](../evidence/issue-020/spec-authority-2026-04-30.md) | RFC 7807 §3 ("identify the problem"). No strict CSAPI spec violation; the binding rule is the RFC 7807 problem-detail principle plus the maintainer's own post-`1b2b614` pattern of distinct messages per case. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted issues #19 + #20 (folded
together) and shipped the disentangled-error-messages pattern in
`1b2b614`. Implications:

- Tone: **closing the last gap in `1b2b614`'s coverage**. Frame as
  "the empty-string case (T5 in the original matrix) was not given
  its own branch in the fix and still emits the `is required`
  message. One additional branch with a distinct message closes
  the matrix."
- The fix shape is unambiguous — copy-paste an additional branch
  modeled on the existing wrong-type branch in `1b2b614`. No
  judgment-call.

## 5. Upstream commits relevant to the area

- **`1b2b614`** — *the parent fix.* Applied the
  `present && nil-check && type-assert-with-explicit-error` pattern
  to `phenomenonTime` and `resultTime`, reducing conflation from
  6→1 to 2→1. Cite as the precedent and the file the recommended
  fix would extend. The body's proposed fix code in the issue-020
  eval is the canonical pattern; the recommended fix here is to add
  the single `s == ""` branch that the body included but the
  shipped commit omitted.
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) `1b2b614` is in `upstream/main` lineage;
  (b) `resultTime` and `phenomenonTime` decode blocks have explicit
      branches for wrong-type and (missing/null), but **not** for
      empty-string;
  (c) live POST with empty-string `resultTime` still returns the
      `"resultTime is required"` message.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm 1b2b614 is in main lineage:
git log --oneline upstream/main | Select-String -Pattern '1b2b614'

# Read the parent fix to confirm exact branch shape post-fix:
git show 1b2b614 -- internal/api/observation_handler.go

# Read current decoder and locate the resultTime / phenomenonTime blocks:
git show upstream/main:internal/api/observation_handler.go | Select-String -Pattern 'resultTime|phenomenonTime|decodeError' -Context 6,6

# Confirm the empty-string branch is NOT present for resultTime / phenomenonTime:
git show upstream/main:internal/api/observation_handler.go | Select-String -Pattern 'must be a non-empty|s == ""|rtStr == ""|ptStr == ""' -Context 1,1

# Recent history on observation_handler.go:
git log --oneline upstream/main -- internal/api/observation_handler.go | Select-Object -First 10
```

```pwsh
# Live re-verification — auth required for POST. Sketch:
# T5-resultTime: POST Observation with `"resultTime": ""`
#   Expected: 400 with body `"resultTime is required"` (the misleading message)
# T5-phenomenonTime: POST Observation with valid resultTime + `"phenomenonTime": ""`
#   Expected: 400 with the same misleading message OR silent acceptance,
#   depending on how the post-`1b2b614` phenomenonTime block handles empty.
# Sibling controls (sanity, should already be distinct post-1b2b614):
#   - missing resultTime → 400 "resultTime is required" (correct)
#   - numeric resultTime (T1) → 400 "resultTime must be an RFC 3339 …" (correct)
#   - malformed-string resultTime (T2) → 400 "Invalid resultTime format" (correct)
```

Expected on unchanged post-`1b2b614` state:
- `1b2b614` in `upstream/main` lineage.
- `resultTime` and `phenomenonTime` decode blocks have distinct
  branches for wrong-type and (missing/null), but no `s == ""`
  branch.
- T5 (empty string) live → 400 `"resultTime is required"`.
- Other matrix rows return their own distinct messages (sanity).

If the empty-string branch is already present on `upstream/main`,
**stop drafting** — defect already fixed.

## 7. Spec-authority sources

This finding has no strict CSAPI spec binding (CSAPI doesn't
prescribe error wording). Spec authority comes from the RFC 7807
problem-detail principle plus the maintainer's own
post-`1b2b614` pattern.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **RFC 7807** §3 (problem-detail "identify the problem") | "RFC 7807 — Problem Details for HTTP APIs" under *IETF RFCs* | The binding principle: error responses should identify the specific problem, not collapse multiple distinct client errors into one message. **Primary citation.** |
| **OGC 23-001** — CSAPI Part 1 §"Error responses" | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms CSAPI defers to OGC API – Common / RFC 9457 / RFC 7807 for error-response shape; no override or specialization. |
| **OGC 19-072** — OGC API – Common §error responses | "OGC API - Common - Part 1: Core (OGC 19-072)" under *OGC Common* | Supporting: OGC API – Common adopts RFC 7807 problem-detail semantics. |
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: status code is correct; this filing is purely about message accuracy within that 400 response. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — not the binding standard for error message wording.
- JSON Schema 2020-12 — not relevant.
- RFC 7493 — relevant for parent #19 (data-integrity twin), not for
  this UX residual.

## 8. Open questions

1. **Recommended fix shape** is unambiguous (add the single
   `s == ""` branch with a distinct message). No judgment-call.
2. **Apply to both `resultTime` AND `phenomenonTime`?** Backlog
   says yes (both fields). §6 must confirm the post-`1b2b614`
   `phenomenonTime` block has the same gap (no empty-string
   branch). **Tentative answer: yes, recommend the fix for both
   fields in one PR.**
3. **Re-litigate the null vs. missing collapse?** The eval's
   "Refinement vs. issue body framing" §noted RFC 8259 §3
   distinguishes null from missing but called the collapse
   defensible because the client correction is the same. The
   shipped fix in `1b2b614` followed the body's proposal and
   collapsed them. **Tentative answer: do not re-litigate; this
   filing is exclusively about T5 (empty string).**
4. **Severity P3 vs. P2?** Plan retains P3 per backlog and eval.
   No data-integrity impact; pure UX/error-message issue.
5. **Live reproducer required?** Auth needed for POST. Static
   evidence + the eval's existing 7-row matrix as transitive
   precedent should be sufficient. **Tentative answer: capture
   live opportunistically; static-only acceptable if auth blocks.**
6. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–07.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: `1b2b614` disentangled the conflation matrix in `decodeObservationPayload` from 6→1 to 2→1. The remaining conflation is the empty-string case (T5 in the original 7-row matrix): `"resultTime": ""` passes `(string); ok` but is then skipped by the inner non-empty guard, falls through to the late `IsZero()` rescue, and emits the misleading `"resultTime is required"` message. |
| **Claim** | One bullet: an empty-string `resultTime` (or `phenomenonTime`) is reported as `is required` rather than as a distinct empty-string error. |
| **Static evidence** | Excerpt of the post-`1b2b614` decoder showing the distinct branches for wrong-type and (missing/null) and the absence of a branch for empty-string. From [`evidence/issue-020/static-analysis-2026-04-30.md`](../evidence/issue-020/static-analysis-2026-04-30.md), refreshed in §6. |
| **Live evidence** | If auth permits: T5 POST with `"resultTime": ""` → 400 `"resultTime is required"`; same for `phenomenonTime` (with valid resultTime). Plus 1–2 control rows from the original matrix to demonstrate the other branches now produce distinct messages. If auth blocks: cite the eval's pre-fix matrix as transitive precedent and rely on static evidence. |
| **Spec authority** | One paragraph: RFC 7807 §3 (problem-detail "identify the problem") is the binding principle. CSAPI Part 1 / OGC API – Common defer to RFC 7807. RFC 9110 §15.5.1 confirms 400 is the correct status. No strict CSAPI override. |
| **Recommended fix** | Add a single `if s == "" { return nil, &decodeError{msg: "resultTime must be a non-empty ISO 8601 string"} }` (and equivalent for `phenomenonTime`) between the existing `ok` check and the inner parse block. Mirror the wording style of the existing `1b2b614` messages. |
| **Scope guard** | "What NOT to touch": no change to the wrong-type / missing / null branches (parent fix correct); no re-litigation of the null↔missing collapse; no broader rework of the late `IsZero()` rescue (defense-in-depth, retained per eval); no `samplingFeature@id` work (separate filing — plan-07). |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-08-empty-string-resulttime-phenomenontime.md`](../upstream-issue-reports/report-08-empty-string-resulttime-phenomenontime.md)

Same `NN` and `<slug>`
(`08-empty-string-resulttime-phenomenontime`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
