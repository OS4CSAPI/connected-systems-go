# Research Plan 04 — `SystemEvent` not handled in `SystemRepository.deleteCascade`

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
item **7**, copied verbatim:

> ### 7. `SystemEvent` not handled in `SystemRepository.deleteCascade`
> - **Source:** Issue #16 closure.
> - **Category:** Defect (P2 — orphaning).
> - **Summary:** `fe9fbd0` chose Approach 3b (application-level child checks)
>   over Approach 3a (DB-level FK constraints). Under 3b, every child resource
>   must be enumerated in `deleteCascade`. `SystemEvent` is not. Deleting a
>   system with attached events will succeed and orphan the events rows.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P2 — orphaning**. The eval (issue-016
> §"Bug 3") frames the orphaning channel as **latent P2 / surfaces as
> P1 once Bug 1 is fixed**. Bug 1 is now fixed (commit `fe9fbd0`), so
> the latent channel is **active**. Severity for this filing should be
> **P2** as the backlog says — the underlying defect is the
> Approach-3b enumeration gap, which is a self-contained orphaning
> defect on a narrower blast radius (only systems with attached
> SystemEvents). Plan retains P2.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#16` — closed by upstream commit
  `fe9fbd0` adopting Approach 3b (application-level child checks in
  `deleteCascade`).
- This filing is the **enumeration-gap residual** identified in the
  eval at §"Bug 3": `SystemEvent` is a system child, was not in the
  child-resource set the maintainer enumerated when applying 3b, and
  Approach 3b's correctness depends on completeness of that
  enumeration.
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-016.md`](../issue-evaluations/issue-016.md) | Full evaluation. §"Bug 3" identifies that `SystemRepository.deleteCascade` did not delete `SystemEvent` rows and that this would surface as live orphaning once Bug 1 was fixed. The eval explicitly recommended Approach 3a, but documented 3b as an acceptable alternative *conditional on enumeration completeness*. |
| [`../evidence/issue-016/static-analysis-2026-04-30.md`](../evidence/issue-016/static-analysis-2026-04-30.md) | Source for the "full-file grep returns zero matches for `SystemEvent` / `system_events`" claim. Confirms the missing enumeration is at `internal/repository/system_repository.go` `deleteCascade`. |
| [`../evidence/issue-016/live-test-2026-04-30.md`](../evidence/issue-016/live-test-2026-04-30.md) | Pre-`fe9fbd0` live evidence. Useful for shape comparison only; **fresh live reproducer must be captured during Step 4.2** showing that on post-`fe9fbd0` HEAD a system DELETE with attached SystemEvents either (a) succeeds and orphans, or (b) succeeds because the maintainer's 3b enumeration silently skips them. |
| [`../evidence/issue-016/spec-authority-2026-04-30.md`](../evidence/issue-016/spec-authority-2026-04-30.md) | Background — referential integrity expectations. CSAPI Part 1 / Part 2 do not directly mandate FK behavior on cascade, so spec authority here is **soft**: integrity convention + RFC 7493 robustness + maintainer's own choice of 3b's contract. |
| [`../evidence/issue-002/fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt) | Reused by eval §"Bug 3". Confirms `system_events.system_id` has no DB-level FK, so the orphaning is silent (no `23503` trip). Critical: under Approach 3b without enumeration, the only safety net (the FK) is also absent. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted issue #16 (a P1 umbrella
finding) and shipped Approach 3b in `fe9fbd0`. Implications:

- Tone: **enumeration-completeness follow-up**, not a fresh defect
  report. Frame as "completing the Approach-3b coverage you adopted
  in `fe9fbd0`."
- The maintainer's *choice* of 3b over 3a is what makes this filing
  necessary — under 3a, FK constraints would surface the orphaning
  automatically; under 3b, every child resource must be in the
  enumeration. This is a structural property of the chosen approach,
  not a critique of the choice.

## 5. Upstream commits relevant to the area

- **`fe9fbd0`** — *the parent fix.* Adopted Approach 3b for
  `SystemRepository.deleteCascade` (and the Datastream / ControlStream
  cascade siblings). Cite as both precedent and prior art for the fix
  shape. Re-verification (§6) must read this commit's diff to confirm
  the exact set of children the maintainer enumerated.
- Related but distinct (do **not** bundle here): `parent_deployment_id`
  cascading, system_datastreams / system_controlstreams cleanup — these
  are within `fe9fbd0`'s scope and assumed correct unless §6 finds
  otherwise.
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm fe9fbd0 still in main lineage:
git log --oneline upstream/main | Select-String -Pattern 'fe9fbd0'

# Read the parent fix to enumerate the children it covers:
git show fe9fbd0 -- internal/repository/system_repository.go

# Confirm SystemEvent is still NOT in deleteCascade on current HEAD:
git show upstream/main:internal/repository/system_repository.go | Select-String -Pattern 'deleteCascade|SystemEvent|system_events' -Context 2,2

# Confirm SystemEvent is a child of System in the model:
git show upstream/main:internal/model/domains/system_event.go | Select-String -Pattern 'SystemID|system_id' -Context 1,1

# Confirm system_events.system_id has no DB-level FK (so orphaning is silent):
git show upstream/main:internal/repository/system_event_repository.go | Select-String -Pattern 'foreignKey|references|constraint' -Context 0,0

# Recent history on system_repository.go (look for any post-fe9fbd0 enumeration changes):
git log --oneline upstream/main -- internal/repository/system_repository.go | Select-Object -First 10
```

```pwsh
# Live re-verification — full sequence requires auth. Sketch:
# 1. POST a system
# 2. POST a system_event under that system
# 3. DELETE that system (with cascade if cascade is the documented teardown path)
# 4. Re-query system_events: orphan rows present == defect confirmed
# Capture each curl request/response and paste into the report.
```

Expected on unchanged state:
- `fe9fbd0` is in `upstream/main` lineage.
- `system_repository.go` `deleteCascade` enumerates Deployments,
  Procedures, Datastreams, ControlStreams (and their join tables),
  but **not** `SystemEvent` / `system_events`.
- `SystemEvent` model has `SystemID` (or equivalent) parent reference.
- `system_events.system_id` has no FK constraint.
- Live: DELETE system after attaching a SystemEvent leaves the
  SystemEvent row orphaned in the DB.

If `system_repository.go` `deleteCascade` already enumerates
`SystemEvent` on `upstream/main`, **stop drafting** — defect already
fixed.

If the live test cannot be completed because of auth (likely),
static evidence + the maintainer's own 3b commit diff are sufficient
to file. Note absence of live evidence with the explanation that the
defect is structural and the static evidence is unambiguous.

## 7. Spec-authority sources

This finding is **not** a spec-authority issue in the strict sense —
it is a contract-completeness defect in the maintainer's chosen
Approach 3b. Spec authority plays a corroborating role only.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001** — CSAPI Part 1 §"System events" / SystemEvent association | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms that `SystemEvent` is a first-class child resource of `System` in the spec model — i.e., it's a relationship that the server is responsible for maintaining (so silent orphaning *is* a contract issue, just not via a SHALL clause on cascade). |
| **OGC 23-002** — CSAPI Part 2 SystemEvent schema | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms SystemEvent's parent-system reference. Used to demonstrate the parent-child relationship is normative and not optional. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — not the binding standard for cascade behavior.
- OGC 19-072 — not relevant; this is internal repository behavior.
- RFC 7493 — overreach; this is not a JSON-emission issue.

## 8. Open questions

1. **Frame as "completing 3b" or as "evidence 3b is fragile,
   reconsider 3a"?** The eval explicitly recommended 3a, but the
   maintainer chose 3b. Re-litigating the choice in this filing
   would be hostile and out-of-scope.
   **Tentative answer: frame purely as "completing 3b's
   enumeration"** — narrow, friendly, mechanically simple. Mention
   the 3a alternative only as a footer ("this class of follow-up
   is what backlog item 14 / Approach 3a hardening would prevent
   structurally").
2. **Bundle backlog item 14 (Approach 3a FK tags) into this
   filing?** No — item 14 is conditional on maintainer interest and
   covers a much broader hardening scope. Bundling would muddy the
   pitch.
   **Tentative answer: file separately, conditional on signal
   from this filing's response.**
3. **Should the recommended fix include other enumeration audits
   (e.g., did `fe9fbd0` cover *every* system child)?** §6 will
   confirm exactly which children `fe9fbd0` enumerated. If others
   are also missing, **bundle them all into this filing** (one
   defect per missing child enumeration is excessive churn for a
   structural finding). If only `SystemEvent` is missing, file
   narrowly. **Tentative answer: §6 determines this — drafter must
   read `fe9fbd0` carefully and audit every system-child resource
   defined in `internal/model/domains/`.**
4. **Live reproducer required?** Auth likely blocks the multi-step
   sequence. Static evidence + maintainer's own commit diff are
   sufficient. **Tentative answer: capture live opportunistically
   if auth permits; otherwise note absence with the structural
   argument.**
5. **Cite our eval/evidence paths in the issue body?** Yes, as
   footer "full validation chain" links — same convention as plans
   01–03.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two-three sentences: `fe9fbd0` adopted Approach 3b for delete cascade. Approach 3b's correctness depends on `deleteCascade` enumerating every child resource. `SystemEvent` is a System child per the CSAPI Part 1 / Part 2 spec model and is not enumerated. |
| **Claim** | One bullet: `internal/repository/system_repository.go` `deleteCascade` does not delete `SystemEvent` rows; deleting a system with attached events succeeds and silently orphans the rows because `system_events.system_id` has no DB-level FK. |
| **Static evidence** | Diff or excerpt of `deleteCascade` showing the enumerated children and the absence of `SystemEvent`; one-line confirmation that `system_events.system_id` has no FK; one-line confirmation that `SystemEvent` carries a `SystemID` parent reference. From [`evidence/issue-016/static-analysis-2026-04-30.md`](../evidence/issue-016/static-analysis-2026-04-30.md) and [`evidence/issue-002/fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt). |
| **Live evidence** | If auth permits: POST system → POST event → DELETE system → re-query events showing orphan. If not: cite parent issue #16's live evidence as transitive precedent (same handler, same code path). |
| **Spec authority** | One paragraph: SystemEvent is a normative System child per CSAPI Part 1 §System events. Not a SHALL on cascade per se, but the parent-child relationship is normative, so silent orphaning is a contract violation against the resource model. |
| **Recommended fix** | Single coordinated change: add `SystemEvent` (and any other missing children identified by §6 audit) to `SystemRepository.deleteCascade`, mirroring the existing enumeration pattern (`tx.Where("system_id = ?", id).Delete(&domains.SystemEvent{})`). Single function edit. |
| **Scope guard** | "What NOT to touch": no schema migration; no FK addition (that's backlog item 14, separate filing); no Approach 3b → 3a switch. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-04-systemevent-not-in-deletecascade.md`](../upstream-issue-reports/report-04-systemevent-not-in-deletecascade.md)

Same `NN` and `<slug>`
(`04-systemevent-not-in-deletecascade`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
