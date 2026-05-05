# Upstream Follow-up Plan

Companion to [`upstream-followup-backlog.md`](./upstream-followup-backlog.md).
Defines the action sequence for settling outstanding fork-side issues and
filing the residual backlog upstream against
`SomethingCreativeStudios/connected-systems-go`.

State as of 2026-05-05: 24 of 26 fork issues closed; 2 awaiting maintainer
response (#10 status, #22 pushback); 16-item backlog ready for upstream filing.

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

### Tier A — file immediately after Phase 1 settles (high-confidence P2 defects)

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

### Tier C — file as enhancement bundle (lower priority, optional)

| # | Backlog item | Severity | One-line description |
|---|---|---|---|
| 11 | #15 (backlog) | P3 enh | Inline `@link` absolutization for 5 remaining resource types. |
| 12 | #16 (backlog) | P4 enh | Inline `@link` Type/Title/UID enrichment (per maintainer's *"not all fully enriched"* self-ack). |
| 13 | #6 (backlog)  | enh    | Latent untagged relationship slices on System/Procedure. |
| 14 | #14 (backlog list) — Approach 3a | enh | Schema-side FK tags. **Only if maintainer signals interest** after Tier A item #4 (SystemEvent cascade) lands. |

---

## Phase 3 — Filing protocol per item

**Standing decision (2026-05-05):** All upstream submissions are filed as
**issues**, never as direct pull requests — even for one-line fixes. The
maintainer owns the canonical repo and decides what merges; our job is to
surface findings with full evidence and let them choose the fix shape and
timing. Direct PRs would skip that triage step and risk burning maintainer
trust on unsolicited changes.

For each filing:

1. **Re-verify on `upstream/main` HEAD before filing** — defect may have been
   fixed in passing.
2. **Compose using established issue-template format** with:
   - Context
   - Claim
   - Static evidence (file:line)
   - Live evidence (if applicable)
   - Spec authority (if applicable)
   - Recommended fix
   - Scope guard ("what NOT to touch")
3. **Reference research artifacts**:
   - `docs/research/issue-evaluations/issue-NNN.md`
   - `docs/research/evidence/issue-NNN/*`
   so the maintainer has the full validation chain.
4. **Cross-reference original `OS4CSAPI/connected-systems-go` issue number**
   for traceability back to the closure pass.
5. **After filing**, move backlog item from "Open items" → "Closed /
   superseded" with the new upstream issue number.

---

## Phase 4 — Bundling considerations

Some items naturally combine into a single upstream issue for review
efficiency. Drops upstream issue count from 14 → ~10-11.

| Bundle | Items | Rationale |
|---|---|---|
| **Decoder hygiene** | Backlog #9 + #11 | Both touch `time_range.go` / observation decode. |
| **Handler UUID & decode** | Backlog #6 + #8 | Both relate to handler-entry hygiene and decode safety. |
| **Inline `@link` completion** | Backlog #15 + #16 | Same JSON formatter sites; same maintainer touch zone. |
| **Validator orphans** | Backlog #12 + #13 | Both are "model declares it, GET round-trips it, validator ignores it" with `Constraint` and `Updatable` on the same `DatastreamDataComponent` struct. |

---

## Phase 5 — Closure

After Tier A + B filed:

1. **Update backlog "Process notes"** with cross-references to filed upstream
   issue numbers.
2. **Final review** — confirm every fork-side closed issue references either:
   - a fix (commit SHA), or
   - a not-planned rationale, or
   - a backlog → upstream issue pointer.
3. **Decide on Tier C** — file only if:
   - there's bandwidth, **or**
   - maintainer engagement on Tier A/B is positive.

---

## Open decisions

1. **Filing cadence** — file Tier A immediately as separate issues, or wait
   until #22 settles to bundle context?
2. **Tier C ambition** — file all four, or only #15+#16 (which tie to
   maintainer's own *"not fully enriched"* self-ack and have higher
   acceptance odds)?

## Settled decisions

- **2026-05-05** — All upstream submissions are filed as issues, never
  direct PRs. Rationale recorded in Phase 3.

---

## Cross-references

- Backlog: [`upstream-followup-backlog.md`](./upstream-followup-backlog.md)
- Issue evaluations: [`issue-evaluations/`](./issue-evaluations/)
- Evidence files: [`evidence/`](./evidence/)
- Maintainer triage source: tracked in conversation logs;
  reproduced inline in each closure comment for traceability.
