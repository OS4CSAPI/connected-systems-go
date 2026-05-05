# Research Plan 11 — Inline `@link` absolutization for remaining 5 resource types

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
item **15**, copied verbatim:

> ### 15. Inline `@link` absolutization for remaining 5 resource types
> - **Source:** Issue #24 closure.
> - **Category:** Enhancement (P3 — spec conformance, sibling of fix that landed).
> - **Summary:** `d2d1347` removed `SystemLink` from the Datastream and
>   ControlStream domain models and made the JSON formatter project
>   `system@link` from `SystemID` via `ToFunctionalAssociationHref`, producing
>   absolute URIs. The same wire-format projection pattern was not applied to
>   the other 5 resource types with inline `@link` fields enumerated in the
>   eval: System, Deployment, SamplingFeature, Observation, Command (17
>   inline link properties total across 7 resource types per
>   `internal/model/domains/`). Recommended: audit each of the 5 remaining
>   resource types and apply the same domain-model-removal + formatter-projection
>   treatment where applicable, or accept that some inline `@link` fields are
>   user-supplied (round-trip-preserved) and only normalize on serialize.
> - **Status:** Ready to file after closure pass.

Tier: **B** (enhancement, single-issue scope, broader audit completion).

> **Severity note.** Backlog says **P3 — spec conformance enhancement**.
> Eval (issue-024) confirmed P3 for the parent fix. This filing
> extends the same conformance argument to the remaining 5 resource
> types; severity is identical because the `format: uri` argument
> applies uniformly to every inline `@link` regardless of resource
> type. P3 stands.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#24` — closed by `d2d1347` for
  Datastream + ControlStream. This filing extends the same
  treatment to the other 5 resource types enumerated by the eval.
- Backlog item 16 (plan-12) is the symmetric `Type/Title/UID`
  enrichment filing — same formatter sites, but content addition
  rather than href absolutization. **Sequence after this one** per
  the backlog's guidance ("both touch the same formatter sites").
- No other fork issues bundle in.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-024.md`](../issue-evaluations/issue-024.md) | Full evaluation. §"Affected-fields completeness" enumerates 17 inline link properties total across 7 resource types per `internal/model/domains/`; #24's fix covered DS+CS, leaving 5 resource types unaddressed. The eval's §"Acceptance criterion sharpening" already proposed an `^https?://` regex test across **all** resource types — the audit completion is exactly this filing. |
| [`../evidence/issue-024/static-analysis-2026-04-30.md`](../evidence/issue-024/static-analysis-2026-04-30.md) | Source-of-truth for the 17-property enumeration and the relative-href grep. **Drafter must re-grep `upstream/main` post-`d2d1347`** to confirm which of the 17 sites are now absolutized (the 2 fixed) vs. still relative (the residual ≈15 across 5 resource types). |
| [`../evidence/issue-024/live-test-2026-04-30.md`](../evidence/issue-024/live-test-2026-04-30.md) | Pre-fix matrix is for DS/CS only (T1, T2). **A fresh per-resource-type matrix must be captured during Step 4.2** for at least System, Deployment, SamplingFeature, Observation, Command — POST a representative resource with an inline `@link` and GET to confirm the href is still relative on the wire. |
| [`../evidence/issue-024/spec-authority-2026-04-30.md`](../evidence/issue-024/spec-authority-2026-04-30.md) | OAS31 inline `@link` schema with `href: format: uri` (lines 312-324) reused via `*ref_11`/`*ref_12` for all enumerated inline link properties. Same citation chain applies to all 7 resource types — that's the whole basis for filing the audit completion. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted #24 (P3 enhancement)
and shipped `d2d1347` adopting the eval's recommended **wire-format
projection** approach (model removal + formatter projection from
`SystemID`) for DS/CS. Implications:

- Tone: **completing the audit the maintainer started.** Frame as
  "the eval that informed #24 enumerated 17 inline `@link`
  properties across 7 resource types; `d2d1347` covered 2 of them
  (DS/CS); this issue lists the remaining 5 resource types so the
  audit lands complete."
- The maintainer's own commit pattern is the precedent — drafter
  should reference `d2d1347`'s structural shape (model field
  removed, formatter projects from FK ID via
  `ToFunctionalAssociationHref`) as the template each remaining
  resource type can follow **where applicable**.
- **Important caveat from the backlog:** "or accept that some
  inline `@link` fields are user-supplied (round-trip-preserved)
  and only normalize on serialize." Not every inline `@link` on
  the 5 resource types is a derived FK projection — some may be
  user-supplied associations that must be persisted as written.
  For those, the alternative fix is normalize-on-serialize via
  `ToFunctionalAssociationHref`'s idempotency on absolute inputs.
  Drafter must distinguish the two cases per resource type.

## 5. Upstream commits relevant to the area

- `d2d1347` — closing commit for #24. Removed `SystemLink` from
  Datastream + ControlStream domain models and added formatter
  projection from `SystemID`. Cite as **the structural precedent**
  for derived inline links.
- The `ToFunctionalAssociationHref` helper at
  `internal/model/formaters/association_links.go:371-389` —
  idempotent on absolute inputs (per the issue-024 eval). Cite as
  **the safe normalization primitive** for the user-supplied
  inline-link case.
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm:
  (a) DS+CS inline `system@link` now emits absolute (parent fix in
      place);
  (b) the remaining 5 resource types still emit relative inline
      `@link.href` values.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm parent fix d2d1347 is in upstream/main:
git log --oneline upstream/main | Select-String -Pattern 'd2d1347|@link|absolut'

# Re-enumerate inline @link properties across all 7 resource types:
git grep -nE '@link|Link\b.*json:' upstream/main -- internal/model/domains/

# Find every site that constructs a relative href for an inline link:
git grep -nE 'Href:\s*"[^h"]' upstream/main -- internal/api/ internal/model/

# Confirm DS+CS now use ToFunctionalAssociationHref (parent fix landed):
git grep -nE 'ToFunctionalAssociationHref' upstream/main -- internal/api/ internal/model/formaters/

# Find handler-synthesis sites for the 5 remaining resource types:
git grep -nE 'SamplingFeature|Deployment|Observation|Command|System(?!Repository)' upstream/main -- internal/api/ | Select-String -Pattern 'Link\{|\.Href'

# Confirm the helper is still idempotent on absolute inputs (read source):
git show upstream/main:internal/model/formaters/association_links.go | Select-String -Pattern 'IsAbs|parsed' -Context 3,3
```

```pwsh
# Live re-verification — auth required for POST. Per-resource-type matrix:
# For each of: System, Deployment, SamplingFeature, Observation, Command:
#   1. POST a minimal valid resource with an inline @link field set to a
#      relative href (or omit and let server synthesize).
#   2. GET the resource — capture the inline @link.href value.
#   3. Record: relative or absolute?
# 4. Cross-check: in the same response, the supplementary links[] array
#    should be absolute (server-side, the same baseURL helper applies there).
#    Asymmetry between inline @link.href and links[i].href confirms the
#    defect on that resource type.
```

Expected on unchanged state:
- DS+CS: inline `system@link.href` absolute (parent fix in place).
- 5 remaining resource types: inline `@link.href` properties still
  emit relative URIs (or user-supplied raw values without
  normalization on serialize).
- Helper still idempotent on absolute inputs.

If any of the 5 are already fixed on `upstream/main`, **scope this
filing to whichever still need it** — not all 5 may still be open.

## 7. Spec-authority sources

Direct binding via OAS31 inline `@link` schema; identical to the
parent #24 citation chain.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001 / 23-002** — CSAPI Part 1 + Part 2 OAS31 schemas, inline `@link` Link object with `href: format: uri` | "OGC API - Connected Systems - Part 1: Core" and "Part 2: Dynamic Data" / "Part 1/2: OpenAPI Specification" under *OGC CSAPI Standards* | **Primary citation.** Same `*ref_11`/`*ref_12` reused across all 7 resource types' inline link properties. The audit-completion argument is exactly that the spec citation applies uniformly. |
| **RFC 3986** §4.3 — Absolute URI definition | "RFC 3986 — Uniform Resource Identifier (URI): Generic Syntax" under *IETF RFCs* | Defines what `format: uri` means in JSON Schema 2020-12 / OAS31 context (absolute URI per §4.3, distinct from `format: uri-reference` per §4.1). |
| **OpenAPI 3.1 / JSON Schema 2020-12** — `format` annotation vs. assertion behavior | "OpenAPI Specification 3.1.0" under *API Specification Standards* | Supporting nuance: `format: uri` is annotation-only by default; strict format-assertion validators flag relative values. The eval already calibrated this as a "may flag" hedge. |
| **CSAPI worked examples** — Datastream (lines 1922-1929), Command (lines 3547-3550) | Same OGC CSAPI Part 1/2 OAS31 spec entries above | Worked examples uniformly use absolute `https://...` URIs across all resource types. Cite as the spec authors' demonstrated intent. |

**Out-of-scope sources (do not cite):**
- HTTP semantics RFCs — relative URI resolution is not at issue
  here; this is about wire-format conformance to the OAS31 schema.
- RFC 8288 (Web Linking) — applies to the supplementary `links[]`
  array, which is already absolute and not the subject of this
  filing.

## 8. Open questions

1. **Per-resource-type fix shape: derived vs. user-supplied.** The
   backlog explicitly flags two valid approaches: (a) repeat
   `d2d1347`'s pattern (remove the field from the domain model,
   project from a FK ID) — appropriate for **derived** inline
   links; (b) normalize-on-serialize via `ToFunctionalAssociationHref`'s
   idempotency — appropriate for **user-supplied** inline links
   that must round-trip-preserve. **Tentative answer: drafter
   classifies each of the 5 resource types' inline link properties
   per category in §6, and the issue body recommends the
   appropriate fix per property — not a one-size-fits-all
   prescription.**
2. **Bundle with plan-12 (`Type/Title/UID` enrichment)?** No — the
   backlog's process notes explicitly sequence #15 before #16
   ("both touch the same formatter sites"), which implies separate
   filings. Mention plan-12 as a "see related" footer once filed.
   **Tentative answer: separate filing; cross-reference plan-12
   in scope-guard.**
3. **Issue body length.** Five resource types × possibly multiple
   inline link properties each could yield a long enumeration.
   **Tentative answer: structure as one table per resource type
   with columns `inline-link property | derived-or-user-supplied
   | recommended fix shape`. Keep prose short; the table carries
   the audit content.**
4. **Severity P3 vs. P4?** Plan retains P3 per backlog. Same
   `format: uri` conformance argument as the parent issue. P3
   stands.
5. **Live reproducer required?** Auth needed for POST per
   resource type, and 5 resource types × multiple link
   properties is a lot of matrix coverage. **Tentative answer:
   capture live for the most prominent property per resource
   type (e.g. `System.parentSystem@link`,
   `Deployment.parentDeployment@link`,
   `SamplingFeature.sampledFeature@link`,
   `Observation.datastream@link` if applicable,
   `Command.controlStream@link` if applicable); static evidence
   is sufficient for the rest.**
6. **Cite our eval/evidence paths in the issue body?** Yes — same
   convention as plans 01–10.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: `d2d1347` closed #24 by removing `SystemLink` from Datastream + ControlStream domain models and projecting `system@link.href` from `SystemID` via `ToFunctionalAssociationHref`, yielding absolute URIs. The eval that informed #24 enumerated 17 inline `@link` properties across 7 resource types; the parent fix covered 2, leaving 5 resource types' inline `@link` emissions still relative. |
| **Claim** | One bullet per resource type, summarizing affected inline-link properties (table format per §8.3). |
| **Static evidence** | Refreshed enumeration from §6: `git grep -nE '@link\|Link.*json:' upstream/main -- internal/model/domains/` for the 17-property inventory; `git grep -nE 'Href:\s*"[^h"]'` for the relative-href construction sites. From [`evidence/issue-024/static-analysis-2026-04-30.md`](../evidence/issue-024/static-analysis-2026-04-30.md). |
| **Live evidence** | Per-resource-type T1-shape rows for at least one prominent inline link per remaining resource type (5 rows minimum), demonstrating relative href on the wire and asymmetry vs. supplementary `links[]` entries. |
| **Spec authority** | One paragraph: same OAS31 `*ref_11`/`*ref_12` citation as #24; the schema's reuse pattern is exactly why the audit completion is necessary — the spec applies uniformly. RFC 3986 §4.3 absolute-URI definition; CSAPI worked examples uniformly absolute. |
| **Recommended fix** | Per-resource-type table: derived links → repeat `d2d1347`'s pattern (model removal + formatter projection from FK ID); user-supplied links → normalize-on-serialize via `ToFunctionalAssociationHref`'s idempotency. The eval's "test asserting `^https?://` for **every** inline `@link.href` in fixtures of every resource type" remains the recommended acceptance criterion. |
| **Scope guard** | "What NOT to touch": no change to the supplementary `links[]` array (already absolute); no `Type`/`Title`/`UID` enrichment (separate filing — plan-12); no helper changes (`ToFunctionalAssociationHref` is correct as-is — leverage its idempotency); no schema migration. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-11-inline-link-absolutization-remaining-5.md`](../upstream-issue-reports/report-11-inline-link-absolutization-remaining-5.md)

Same `NN` and `<slug>`
(`11-inline-link-absolutization-remaining-5`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
