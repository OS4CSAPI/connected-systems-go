# Research Plan 07 — `samplingFeature@id` retains silent-drop type assertion

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
item **10**, copied verbatim:

> ### 10. `samplingFeature@id` retains silent-drop type assertion
> - **Source:** Issue #19 closure (adjacent finding).
> - **Category:** Defect (P3 — narrow).
> - **Summary:** `internal/api/observation_handler.go` still decodes
>   `samplingFeature@id` with `if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != ""`.
>   A JSON number for that field is silently dropped. Narrower than the
>   phenomenonTime/resultTime defect (no repository default-fill, so GET
>   shows null rather than a plausibly-correct substitution), but same bug
>   shape. Apply the `present && nil-check && type-assert-with-explicit-error`
>   pattern that phenomenonTime/resultTime now use post-`1b2b614`.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P3 — narrow**. Eval (issue-019
> "Adjacent finding") explicitly notes the narrower blast radius
> compared to parent #19: no repository default-fill means a silent
> drop produces a `null` GET response (visible to the client), not a
> plausibly-correct substitution. P3 stands.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#19` — closed by upstream commit
  `1b2b614` applying the
  `present && nil-check && type-assert-with-explicit-error` pattern
  to `phenomenonTime` and `resultTime` decoding.
- This filing is the **`samplingFeature@id` residual** identified in
  the eval as an adjacent finding: same decoder file, same bug shape,
  not covered by the parent-fix commit.
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-019.md`](../issue-evaluations/issue-019.md) | Full evaluation. "Adjacent finding (not in original issue body) — `samplingFeature@id`" §documents the same `raw[...].(string); ok` pattern at `observation_handler.go:217` (eval-time line — drift expected post-`1b2b614`; §6 must re-locate). "Suggested fix — extension of body's proposal" §3 explicitly recommends applying the same shape to `samplingFeature@id` for consistency. |
| [`../evidence/issue-019/static-analysis-2026-04-30.md`](../evidence/issue-019/static-analysis-2026-04-30.md) | Source for the type-assertion-pattern citations. Documents the symmetric structure of the original three decoder blocks (`phenomenonTime`, `resultTime`, `samplingFeature@id`) — parent #19 fixed two of three. |
| [`../evidence/issue-019/live-test-2026-04-30.md`](../evidence/issue-019/live-test-2026-04-30.md) | Pre-#19-fix reproduction for `phenomenonTime` and `resultTime`. **Post-#19-fix live reproducer for `samplingFeature@id` must be captured during Step 4.2** — POST an Observation with body containing `"samplingFeature@id": 12345` (numeric) and confirm 201 + GET response shows `samplingFeature@id` is null/absent (silent drop). |
| [`../evidence/issue-019/spec-authority-2026-04-30.md`](../evidence/issue-019/spec-authority-2026-04-30.md) | CSAPI Part 1 §5.1 + RFC 7493 §3.4 require rejection of wrong-typed inputs. The spec-authority binding for parent #19 also covers this finding — `samplingFeature@id` is a string ID reference per the Observation schema. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted issue #19 (a P2
data-integrity defect) and shipped the explicit-type-check pattern in
`1b2b614`. Implications:

- Tone: **completing the type-check coverage** for the third decoder
  block in the same file. Frame as "applying the same pattern you
  used in `1b2b614` to `samplingFeature@id`, the third field with
  the same decode shape."
- No judgment-call needed on fix shape — `1b2b614` is the precedent
  and the recommended fix is byte-for-byte the same pattern.

## 5. Upstream commits relevant to the area

- **`1b2b614`** — *the parent fix.* Applied
  `present && nil-check && type-assert-with-explicit-error` to
  `phenomenonTime` and `resultTime` in `observation_handler.go`.
  Cite as the precedent and the file the recommended fix would
  extend.
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) `1b2b614` is in `upstream/main` lineage;
  (b) `phenomenonTime` and `resultTime` decode blocks now use the
      explicit-error pattern (sanity);
  (c) `samplingFeature@id` decode still uses the legacy
      `raw[...].(string); ok && != ""` pattern.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm 1b2b614 is in main lineage:
git log --oneline upstream/main | Select-String -Pattern '1b2b614'

# Read the parent fix to confirm exact pattern shape:
git show 1b2b614 -- internal/api/observation_handler.go

# Read current decoder and locate the samplingFeature@id block:
git show upstream/main:internal/api/observation_handler.go | Select-String -Pattern 'samplingFeature@id|phenomenonTime|resultTime' -Context 4,4

# Confirm the legacy pattern is still present for samplingFeature@id:
git show upstream/main:internal/api/observation_handler.go | Select-String -Pattern 'raw\["samplingFeature@id"\]\.\(string\)' -Context 2,2

# Recent history on observation_handler.go:
git log --oneline upstream/main -- internal/api/observation_handler.go | Select-Object -First 10
```

```pwsh
# Live re-verification — auth required for POST. Sketch:
# POST an Observation with body:
# { ..., "samplingFeature@id": 12345, ... }
# Expected: 201 Created (silent drop), then GET the observation —
# samplingFeature@id absent or null in response (no repository
# default-fill, so no substitution).
# Sibling control: POST with "samplingFeature@id": "valid-string-id" → 201, GET shows the string back.
```

Expected on unchanged post-`1b2b614` state:
- `1b2b614` in `upstream/main` lineage.
- `phenomenonTime` and `resultTime` decode blocks now use the
  `present && nil-check && type-assert-with-explicit-error` pattern.
- `samplingFeature@id` decode still uses
  `if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != ""`.
- Live: numeric `samplingFeature@id` → 201 + GET shows null/absent
  field; string control round-trips correctly.

If `samplingFeature@id` already uses the explicit-error pattern on
`upstream/main`, **stop drafting** — defect already fixed.

If live POST with numeric `samplingFeature@id` returns 400, parent fix
may have been extended; verify via `git log` and stop if so.

## 7. Spec-authority sources

This finding has the same spec posture as parent #19. Direct binding
via RFC 7493 + CSAPI Part 1.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001** — CSAPI Part 1 §5.1 / Observation schema, `samplingFeature@id` field | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms `samplingFeature@id` is typed as a string ID reference in the canonical Observation schema. **Primary citation** — establishes that a JSON number is wrong-typed. |
| **OGC 23-002** — CSAPI Part 2 Observation schema | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Corroborates the string-typed schema for `samplingFeature@id`. |
| **RFC 7493** §3.4 (I-JSON wrong-type rejection) | "RFC 7493 — The I-JSON Message Format" under *IETF RFCs* | The robustness rule: implementations should reject inputs that don't conform to the schema rather than silently coerce or drop. Same citation as parent #19. |
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: malformed request body → 400 with informative message, not silent acceptance. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — not the binding standard for body-content validation.
- JSON Schema 2020-12 — covered transitively via OGC 23-001/23-002 schemas.
- SWE Common JSON encoding — relevant for the parent #19 (datatype
  fields in the Datastream's data component), not for the
  identifier-reference field handled here.

## 8. Open questions

1. **Recommended fix shape** is unambiguous (mirror `1b2b614`'s
   pattern byte-for-byte for `samplingFeature@id`). No judgment-call.
2. **Bundle other ID-reference fields with the same pattern?** If §6
   finds additional `raw["...@id"].(string); ok` sites in
   `observation_handler.go` or sibling handlers (`command_handler.go`,
   `system_event_handler.go`), bundle them all into this filing under
   "the third+ silent-drop type-assertion in the same family."
   **Tentative answer: §6 must `git grep` for the broader pattern.
   If the matches are within `observation_handler.go` or its
   immediate siblings, bundle. If they're in unrelated handlers, file
   separately.**
3. **Severity P3 vs. P2?** Plan retains P3 per backlog and eval. The
   eval explicitly notes the narrower blast radius (no
   default-fill → null in GET → client-visible). P3 stands.
4. **Live reproducer required?** Auth needed for POST. Static
   evidence + clear pattern-match argument should be sufficient.
   **Tentative answer: capture live opportunistically; static-only
   acceptable.**
5. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–06.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: `1b2b614` applied the `present && nil-check && type-assert-with-explicit-error` pattern to `phenomenonTime` and `resultTime` in `observation_handler.go`. The third decoder block in the same function — `samplingFeature@id` — was not covered and retains the legacy `raw[...].(string); ok && != ""` pattern. |
| **Claim** | One bullet: a JSON number for `samplingFeature@id` on `POST /observations` is silently dropped; the row is created with no sampling-feature association and the GET response shows the field as null/absent. |
| **Static evidence** | Excerpt of the current `samplingFeature@id` decode block; side-by-side with the post-`1b2b614` `phenomenonTime` decode block to show the asymmetry. From [`evidence/issue-019/static-analysis-2026-04-30.md`](../evidence/issue-019/static-analysis-2026-04-30.md), refreshed in §6. |
| **Live evidence** | If auth permits: POST with numeric `samplingFeature@id` → 201 + GET shows null; control with string → round-trips. If not: cite the static asymmetry plus parent #19's live evidence as transitive precedent. |
| **Spec authority** | One paragraph: CSAPI Part 1 §5.1 schemas type `samplingFeature@id` as a string ID reference. RFC 7493 §3.4 requires rejection of wrong-typed inputs. RFC 9110 §15.5.1 supports the 400 response shape. Same spec posture as parent #19. |
| **Recommended fix** | Apply `1b2b614`'s pattern byte-for-byte to the `samplingFeature@id` decode block. Single-block edit in the same function the maintainer just touched. Include §6-identified bundle items if any. |
| **Scope guard** | "What NOT to touch": no change to `phenomenonTime` or `resultTime` blocks (parent fix in place); no broader observation validation rework; no schema changes. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-07-samplingfeature-id-silent-drop.md`](../upstream-issue-reports/report-07-samplingfeature-id-silent-drop.md)

Same `NN` and `<slug>`
(`07-samplingfeature-id-silent-drop`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
