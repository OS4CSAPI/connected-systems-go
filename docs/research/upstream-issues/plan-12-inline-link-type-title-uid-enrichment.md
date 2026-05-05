# Research Plan 12 — Inline `@link` Type/Title/UID enrichment (residual from #25)

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
item **16**, copied verbatim:

> ### 16. Inline `@link` Type/Title/UID enrichment (residual from #25)
> - **Source:** Issue #25 closure. Maintainer self-acknowledged: *"Mostly
>   there for most associations however not all fully enriched"*.
> - **Category:** Enhancement (P4 — UX, no spec violation).
> - **Summary:** `3fa1b0c` and `704a9e3` populated server-generated `Rel`
>   (`ogc-rel:*` vocabulary) on the supplementary `links[]` array.
>   `d2d1347` reorganized inline `@link` as a wire-format projection
>   carrying `Href` only. Three enrichment gaps remain on the inline
>   `@link` emission across DS/CS and the other 5 resource types
>   (System, Deployment, SamplingFeature, Observation, Command):
>   (a) `Type` constants — cheap, no DB cost (e.g.
>       `system@link → application/geo+json`,
>       `procedure@link → application/sml+json`).
>   (b) `Title` (= linked-resource Name) — requires read-time enrichment
>       via batched preload in `SerializeAll` to avoid N+1.
>   (c) `UID` (= linked-resource UID) — same enrichment path as Title.
>   Sequence after item #15 (broader 5-resource-type absolutization
>   audit) since both touch the same formatter sites.
> - **Status:** Ready to file after closure pass.

Tier: **B** (enhancement, single-issue scope, content enrichment companion to plan-11).

> **Severity note.** Backlog says **P4 — UX, no spec violation**. Eval
> (issue-025) confirmed P4 for the parent fix. All enriched fields
> are spec-defined as **optional** (no required-when-known clause),
> so non-population is conformant. P4 stands.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#25` — closed by `3fa1b0c` and
  `704a9e3` (supplementary `links[]` `Rel` population) and partly
  superseded by `d2d1347` (inline `@link` reorganized as wire-format
  projection carrying `Href` only). Maintainer self-acknowledged
  the residual: *"Mostly there for most associations however not
  all fully enriched"*. This filing closes the residual.
- **Sequenced after plan-11** (backlog item 15) per the backlog's
  process note: "both touch the same formatter sites." Plan-11 is
  href-correctness; plan-12 is content enrichment on the same
  emission path.
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-025.md`](../issue-evaluations/issue-025.md) | Full evaluation. Three-axis refinement of the body: (a) cheap-vs-expensive split (`Type` at synthesis time, `Title`/`UID` via formatter batched preload to avoid N+1); (b) `rel`-vocabulary consistency note (don't introduce IANA `parent` when the response already uses `ogc-rel:*`); (c) coverage across all 17 inline link properties not just DS+CS. The eval's "Recommended fix scope" §spells out the implementation phasing this filing should describe. |
| [`../evidence/issue-025/static-analysis-2026-04-30.md`](../evidence/issue-025/static-analysis-2026-04-30.md) | Source for the enumeration of inline `@link` synthesis sites and the supplementary `links[]` builder. **Drafter must re-grep `upstream/main` post-`d2d1347`** to capture the current (post-reorg) emission path, since `d2d1347` moved synthesis from handlers to formatters. |
| [`../evidence/issue-025/live-test-2026-04-30.md`](../evidence/issue-025/live-test-2026-04-30.md) | Pre-fix matrix shows: inline `system@link` carries only `href`; supplementary `links[]` carries `href`+`rel`; client-supplied optionals round-trip preserved (T2 already passes — **regression guard, not new test**). **A fresh post-`d2d1347` matrix must be captured during Step 4.2** to confirm `Type`/`Title`/`UID` are still absent on the wire after the reorganization. |
| [`../evidence/issue-025/spec-authority-2026-04-30.md`](../evidence/issue-025/spec-authority-2026-04-30.md) | OAS31 lines 312-372 define `Href` (required) + `Rel`/`Type`/`Title`/`UID` (all optional). Authority for "all enrichments are spec-defined; populating them is purely additive, no breakage risk." |

## 4. Maintainer triage signal

**Strongly positive with explicit acknowledgment.** Maintainer
already self-acknowledged the residual on #25's closure (*"Mostly
there for most associations however not all fully enriched"*) and
shipped both the `Rel` population (`3fa1b0c`/`704a9e3`) and the
inline `@link` reorganization (`d2d1347`). Implications:

- Tone: **closing out a maintainer-acknowledged residual.** Frame
  as "tracking the residual the maintainer flagged on #25's
  closure to a discrete issue with a phased implementation plan."
  Quote the maintainer's acknowledgment in the issue body.
- Structural precedent is strong: the response already populates
  `Rel` via `ogc-rel:*` on `links[]`, and the eval's
  cheap-vs-expensive split is well-grounded. Drafter should
  reference both prior commits as the implementation pattern.
- **Important: `d2d1347` moved the emission path.** Plan-11's
  drafter is doing exactly this re-grep; if plan-11 is filed
  first, plan-12's §6 should reference plan-11's findings rather
  than re-do the work.

## 5. Upstream commits relevant to the area

- `3fa1b0c` and `704a9e3` — supplementary `links[]` `Rel`
  population. Cite as the **rel-vocabulary precedent** (`ogc-rel:*`
  used here is the convention the inline `@link` enrichment
  should align to, NOT IANA `parent`).
- `d2d1347` — inline `@link` reorganized as wire-format
  projection. Cite as **the current emission site** (the formatter,
  not the handler/repository synthesis sites the issue-025 body
  references).
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) `Rel` population on supplementary `links[]` still in place
      (parent fix in place);
  (b) inline `@link` still emits `Href` only post-`d2d1347` (no
      `Type`/`Title`/`UID`).

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm parent fixes (3fa1b0c, 704a9e3, d2d1347) are in upstream/main:
git log --oneline upstream/main | Select-String -Pattern '3fa1b0c|704a9e3|d2d1347|@link|enrich|rel'

# Locate current inline @link emission sites (post-d2d1347 reorg):
git grep -nE '@link|FunctionalAssociation|ToFunctionalAssociationHref' upstream/main -- internal/model/formaters/

# Confirm supplementary links[] Rel population still in place:
git grep -nE 'ogc-rel:|appendDatastreamAssociationLinks|appendControl' upstream/main -- internal/model/formaters/

# Inventory all 17 inline link properties across 7 resource types:
git grep -nE '@link.*json:|json:".*@link"' upstream/main -- internal/model/

# Find Link struct definition (confirm Type/Title/UID fields exist):
git show upstream/main:internal/model/common_shared/links.go | Select-String -Pattern 'type Link|Href|Rel|Type|Title|UID' -Context 1,1

# Find existing batched preloads to use as N+1 avoidance precedent:
git grep -nE 'Preload|SerializeAll' upstream/main -- internal/model/formaters/ internal/repository/
```

```pwsh
# Live re-verification — auth required. Per-resource-type matrix:
# For each of: Datastream, ControlStream, System, Deployment, SamplingFeature,
#              Observation, Command:
#   1. POST a minimal valid resource that has at least one inline @link.
#   2. GET the resource — capture each inline @link object verbatim.
#   3. Record presence/absence of: type, title, uid (href is plan-11's scope).
# Cross-check: client-supplied optional fields (T2 in live-test) MUST still
# round-trip preserved — this is the regression guard, not new test scope.
```

Expected on unchanged state:
- Supplementary `links[]` array carries `href`+`rel` (parent fix in
  place; `ogc-rel:*` vocabulary).
- Inline `@link` objects on all 7 resource types carry only `href`
  post-`d2d1347` — `type`, `title`, `uid` absent.
- `Link` struct already supports all five optional fields.
- Client-supplied optionals still round-trip preserved on POST→GET.

If `Type` is already populated (cheap pass landed), **scope this
filing to whatever residual remains** (`Title`/`UID` only).

## 7. Spec-authority sources

Direct binding via OAS31 inline `@link` schema (same chain as #24
and #25); identical citation set as plan-11.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001 / 23-002** — CSAPI Part 1 + Part 2 OAS31 schemas, Link object (`href` required + `rel`/`type`/`title`/`uid` optional, OAS31 lines 312-372) | "OGC API - Connected Systems - Part 1: Core" and "Part 2: Dynamic Data" / "Part 1/2: OpenAPI Specification" under *OGC CSAPI Standards* | **Primary citation.** All four enriched fields are spec-defined; none are required-when-known. Authority for "purely additive, no client breakage." |
| **OGC functional-association rel vocabulary** (`ogc-rel:host`, `ogc-rel:systems`, etc.) | Same OGC CSAPI Part 1/2 spec entries above | Authority for the `rel` vocabulary cs-go's `links[]` already uses (`3fa1b0c`/`704a9e3`). The inline `@link` enrichment must align to this convention, not introduce IANA `parent`. |
| **MIME type registry** — `application/geo+json` (RFC 7946), `application/sml+json` (OGC SensorML JSON encoding) | "RFC 7946 — The GeoJSON Format" under *IETF RFCs*; OGC SensorML entry under *OGC SWE Common Standards* (drafter must verify the exact reference-list entry) | Authority for the `Type` constants per resource type — System/Deployment/SamplingFeature → `application/geo+json`; Procedure → `application/sml+json`. |
| **OpenAPI 3.1 / JSON Schema 2020-12** — `format` annotation behavior | "OpenAPI Specification 3.1.0" under *API Specification Standards* | Supporting: confirms that absent optional fields are spec-conformant (P4 not P3). |

**Out-of-scope sources (do not cite):**
- IANA Link Relations registry — explicitly NOT the convention to
  use here; the eval's refinement §3 calls this out as a
  vocabulary-consistency hazard.
- RFC 8288 (Web Linking) — applies to the supplementary `links[]`
  array, not inline `@link`.

> **Note on SensorML / `application/sml+json` reference-list
> entry.** Drafter must verify the SensorML JSON encoding entry
> exists in the curated references list. Surface to user if
> missing — the `procedure@link → application/sml+json` Type
> recommendation requires a normative source.

## 8. Open questions

1. **Phasing — single PR or two?** Eval recommends a phased
   implementation: cheap pass (`Type` constants, no DB cost) first,
   enrichment pass (`Title`/`UID` via batched preload) second.
   **Tentative answer: file as one issue describing both phases;
   maintainer can split into two PRs if desired. Issue body
   structure: §"Cheap pass" + §"Enrichment pass" with
   independent acceptance criteria each.**
2. **`Rel` on inline `@link` — populate or omit?** Eval flagged
   this: the property name (`system@link`) already conveys the
   relation, and the supplementary `links[]` already carries the
   formal `ogc-rel:*` rel. Three options: (a) omit `Rel` from
   inline `@link`; (b) populate matching `ogc-rel:*` on inline
   `@link`; (c) introduce IANA `parent`. **Tentative answer:
   recommend option (a) — omit from inline `@link` — because the
   property name conveys the relation and avoids vocabulary
   duplication. Note (b) as acceptable alternative; explicitly
   reject (c) per eval's vocabulary-consistency argument.**
3. **N+1 avoidance pattern.** Batched preload joining linked
   resources by FK ID for a page of items is the standard
   approach. Plan instructs §6 to find existing precedent in the
   repo (likely `Preload` calls in repositories). **Tentative
   answer: defer line-level implementation to fix PR; issue body
   states the requirement ("MUST avoid N+1") and references any
   precedent §6 surfaces.**
4. **Coverage scope.** All 17 inline link properties across all 7
   resource types — same audit completion shape as plan-11. Do
   NOT scope-creep to supplementary `links[]` (parent `Rel` fix
   already shipped). **Tentative answer: explicit scope guard;
   inline `@link` only.**
5. **Severity P4 vs. P3?** Plan retains P4 per backlog and eval.
   No spec violation; purely UX. P4 stands.
6. **Live reproducer required?** Auth needed; 7 resource types ×
   multiple inline links is a lot. **Tentative answer: capture
   live for one prominent inline link per resource type (same
   strategy as plan-11); static-only acceptable for the rest.
   Regression guard on T2 (round-trip preservation) is critical
   and must be exercised.**
7. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–11.
8. **Cross-reference plan-11 in scope-guard.** Yes — both issues
   touch the same formatter sites; reviewers should see them
   sequenced. **Tentative answer: footer note pointing to
   plan-11's filed issue once known.**

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Three sentences: `3fa1b0c`/`704a9e3` populated `Rel` on supplementary `links[]`; `d2d1347` reorganized inline `@link` as wire-format projection carrying `Href` only; maintainer self-acknowledged on #25's closure that "not all fully enriched". This filing tracks the residual three enrichment gaps (`Type`, `Title`, `UID`) across all 17 inline link properties on all 7 resource types. |
| **Claim** | Three bullets, one per gap: (a) `Type` constants absent on every server-synthesized inline `@link`; (b) `Title` (= linked-resource Name) absent; (c) `UID` (= linked-resource UID) absent. All optional per spec; non-population is conformant but loss-of-information for clients. |
| **Static evidence** | Refreshed enumeration from §6: 17 inline link properties; current emission site post-`d2d1347` (formatter); `Link` struct already supports all five fields. From [`evidence/issue-025/static-analysis-2026-04-30.md`](../evidence/issue-025/static-analysis-2026-04-30.md). |
| **Live evidence** | Per-resource-type matrix (one prominent inline link per resource type, 7 rows) showing absent `type`/`title`/`uid` on the wire. Plus T2-shape regression-guard row showing client-supplied optionals still round-trip preserved (already passing). |
| **Spec authority** | One paragraph: OAS31 lines 312-372 define all four fields as optional (`href` required); RFC 7946 / OGC SensorML define the MIME-type constants for `Type`; cs-go's existing `ogc-rel:*` vocabulary on `links[]` is the rel-naming precedent if `Rel` is populated. P4: purely additive, no breakage risk. |
| **Recommended fix** | Two-phase plan from eval: §"Cheap pass" — populate `Type` at the formatter projection site (constant per resource type), no DB cost; §"Enrichment pass" — populate `Title` and `UID` via batched preload in `SerializeAll` to avoid N+1; mirror across all 7 resource types' formatters. Acceptance criteria stated independently per phase so they can ship as one or two PRs. |
| **Scope guard** | "What NOT to touch": no change to supplementary `links[]` (`Rel` fix already shipped); no inline `@link.href` absolutization (separate filing — plan-11); no introduction of IANA `parent` rel (vocabulary consistency); no Link struct changes (already supports all fields); no client-side round-trip behavior changes (regression guard). |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-12-inline-link-type-title-uid-enrichment.md`](../upstream-issue-reports/report-12-inline-link-type-title-uid-enrichment.md)

Same `NN` and `<slug>`
(`12-inline-link-type-title-uid-enrichment`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
