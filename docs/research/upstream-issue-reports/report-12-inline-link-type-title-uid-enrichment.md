# Report 12 — Inline `@link` Type/Title/UID enrichment (residual from #25)

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-12-inline-link-type-title-uid-enrichment.md`](../upstream-issues/plan-12-inline-link-type-title-uid-enrichment.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#16** (Inline `@link` Type/Title/UID enrichment — residual from #25) |
| Source fork issue | `OS4CSAPI/connected-systems-go#25` (closed by `3fa1b0c` + `704a9e3` + `d2d1347`); maintainer self-acknowledged residual *"Mostly there for most associations however not all fully enriched"*. |
| Research plan | [`../upstream-issues/plan-12-inline-link-type-title-uid-enrichment.md`](../upstream-issues/plan-12-inline-link-type-title-uid-enrichment.md) |
| Companion filing | [`./report-11-inline-link-absolutization-remaining-5.md`](./report-11-inline-link-absolutization-remaining-5.md) — same emission sites, href-correctness axis. |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P4** — UX, no spec violation; all enriched fields are spec-defined as optional. |
| Tier | **B** (single-issue scope) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`.
All three parent commits in lineage: `3fa1b0c` + `704a9e3` (`Rel`
population on supplementary `links[]`) + `d2d1347` (inline
`@link` reorganized as wire-format projection).

**Re-verification revised one boundary of the plan's framing:**
the supplementary `links[]` array is **already richly enriched
beyond `Rel`**. `internal/model/formaters/association_links.go`
populates `Type` (via `GeoJSONContentType` /
`SensorMLContentType` constants), `Title` (= linked-resource
`Name`), and `UID` (= linked-resource `UniqueIdentifier`) on
supplementary `links[]` entries, with a `ResourceCache`
abstraction (`FetchParentSystems`, `FetchProcedures`) that batches
preloads to avoid N+1. **The residual gap is therefore confined
to inline `@link` emission, not the supplementary array.**

```text
$ git show upstream/main:internal/model/common_shared/links.go | head
type Link struct {
    Href  string  `json:"href"`
    Rel   string  `json:"rel,omitempty"`
    Type  string  `json:"type,omitempty"`
    Title string  `json:"title,omitempty"`
    UID   *string `json:"uid,omitempty"`
}
const OGCRelPrefix = "ogc-rel:"
```

```text
$ git grep -nE 'Type:|link.Title|link.UID' upstream/main -- internal/model/formaters/association_links.go
# Demonstrates extensive Type/Title/UID population on supplementary links[]:
internal/model/formaters/association_links.go:137: Type: GeoJSONContentType,
internal/model/formaters/association_links.go:141: link.Title = parent.Name
internal/model/formaters/association_links.go:144: link.UID = &uid
internal/model/formaters/association_links.go:156, 163, 170, 177, 184, 192, 307, 315, 332, 350: Type: GeoJSONContentType / SensorMLContentType
internal/model/formaters/association_links.go:197, 227, 282: link.Title = <resource>.Name
internal/model/formaters/association_links.go:200, 230, 285: link.UID = &uid
internal/model/formaters/association_links.go:361-362: if link.Type == "" { link.Type = GeoJSONContentType }
```

**Inline `@link` projection sites — emission shape post-`d2d1347`:**

```text
$ git show upstream/main:internal/model/formaters/json_formatters/datastream_json.go | grep SystemLink
type DatastreamJSONFeature struct {
    domains.Datastream
    SystemLink *common_shared.Link `json:"system@link,omitempty"`
}
// formatter projection — Href-only construction:
out.SystemLink = &common_shared.Link{
    Href: formaters.ToFunctionalAssociationHref("/systems/" + *datastream.SystemID),
}
// (identical shape in control_stream_json.go)
```

The `SystemLink` literal sets **only `Href`**. `Type`, `Title`,
`UID` (and `Rel`) are all zero-valued and therefore omitted under
the struct's `,omitempty` tags. The same shape applies symmetrically
to the other inline-link sites the report-11 audit enumerated.

**Inline-link inventory (carry-forward from report-11 §1):** 14
distinct inline `@link` properties across 7 resource types (DS,
CS, Command, Observation, Deployment, SamplingFeature, System).
**All emit `Href`-only on the wire post-`d2d1347`** — none
populate `Type`, `Title`, or `UID`.

**Live re-verification:** not captured this round (auth-gated
POST per resource type). Defect is structural — every inline-link
projection site constructs a `common_shared.Link{Href: ...}`
literal with zero-valued enrichment fields, visible in the static
evidence. T2 round-trip-preservation (the regression guard from
issue-025 live evidence) is a separate behavior on the input
deserialization path, unaffected by formatter-side enrichment.

## 2. Static evidence

Source: [`../evidence/issue-025/static-analysis-2026-04-30.md`](../evidence/issue-025/static-analysis-2026-04-30.md)
"Affected fields" §, refreshed in §1 above to reflect the
post-`d2d1347` emission path.

Three independent enrichment gaps, each spec-optional:

### 2.1 `Type` — cheap pass, no DB cost

The `Link.Type` field is constant per inline-link property
(determined by the linked resource type's representation MIME):

| Inline-link property | Linked resource type | Recommended `Type` constant |
|---|---|---|
| `system@link`, `subsystems@link`, `deployedSystems@link`, `parentSystem@link`, etc. | System (GeoJSON-default) | `application/geo+json` |
| `deployment@link`, `parentDeployment@link`, `subdeployments@link` | Deployment (GeoJSON-default) | `application/geo+json` |
| `featureOfInterest@link`, `samplingFeature@link`, `sampledFeature@link` | SamplingFeature (GeoJSON-default) | `application/geo+json` |
| `procedure@link`, `systemKind@link`, `usedProcedures@link` | Procedure / SystemKind (SensorML) | `application/sml+json` |
| `result@link` (Observation) | Free-form per `resultEncoding` | omit `Type`, or set per upstream's existing convention |

Constants `GeoJSONContentType` and `SensorMLContentType` are
already defined in `association_links.go:15-16` and are the
authoritative repo-side names for these MIME values. The cheap
pass is a one-line addition per inline-link projection
(`Type: formaters.GeoJSONContentType,` etc.) at every site
report-11 enumerates.

### 2.2 `Title` — enrichment pass via batched preload

The `Link.Title` field semantically maps to the linked resource's
`Name` (or display label) at the time of serialization.
Population requires a lookup of the linked resource by FK ID. The
**precedent for N+1 avoidance is already in the repo** —
`ResourceCache` in `association_links.go` exposes
`FetchParentSystems` / `FetchProcedures` and is used by the
supplementary `links[]` enrichment path. The same cache (or its
extension to additional resource types) should be threaded into
the inline-link formatter projection sites for `SerializeAll`
batches.

### 2.3 `UID` — enrichment pass via batched preload (same path as Title)

The `Link.UID` field maps to the linked resource's
`UniqueIdentifier`. Identical enrichment path as `Title`; the
existing `ResourceCache` pattern populates them together
(association_links.go:141-144, 197-200, 227-230, 282-285).

### 2.4 What's NOT a gap (carve-outs)

- Supplementary `links[]` enrichment is **already complete** for
  Type/Title/UID where the cache covers the linked resource type
  (System, Deployment via parent traversals, Procedure). Filing
  scope is inline `@link` only.
- `Rel` on inline `@link` — see §5 alternative analysis. Property
  name conveys the relation; `Rel` is omittable and recommended omitted.
- The `Link` struct itself supports all five fields (see §1 head dump).

## 3. Live evidence

Not captured this round. Justification in §1: defect is
structural (every projection site constructs `Link{Href: ...}`
with zero-valued enrichment fields), and parent #25's pre-fix
matrix transitively applies to the post-`d2d1347` emission path
since the reorganization moved synthesis from
handler/repository to formatter without altering the field-set
populated.

The plan's per-resource-type matrix sketched in §6 follows
deterministically: 7 resource types × 3 enrichment fields = 21
expected-absent cells; T2 (round-trip preservation of
client-supplied optionals on input) is unaffected and remains
the regression guard.

## 4. Spec authority

Identical citation chain to #25 — all four enriched fields are
spec-defined optional, so the filing is purely additive.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001 / 23-002** — CSAPI Part 1 + Part 2 OAS31 schemas, `Link` object (`href` required + `rel`/`type`/`title`/`uid` optional, OAS31 lines 312-372) | "OGC API - Connected Systems - Part 1: Core" and "Part 2: Dynamic Data" / "Part 1/2: OpenAPI Specification" under *OGC CSAPI Standards* | **Primary citation.** All four enriched fields are spec-defined; none are required-when-known. Authority for "purely additive, no client breakage." |
| **OGC functional-association `rel` vocabulary** (`ogc-rel:host`, `ogc-rel:systems`, etc.) | Same OGC CSAPI Part 1/2 spec entries above | Vocabulary already adopted by upstream's supplementary `links[]` (`3fa1b0c`/`704a9e3`); cited only to reject IANA `parent` if `Rel` on inline `@link` is debated. |
| **RFC 7946** — The GeoJSON Format (`application/geo+json` MIME) | "RFC 7946 — The GeoJSON Format" under *IETF RFCs* | Authority for the `Type` constant `application/geo+json` for System / Deployment / SamplingFeature inline links. |
| **OGC SensorML JSON Encoding** (`application/sml+json` MIME) | OGC SensorML entry under *OGC SWE Common Standards* | Authority for the `Type` constant `application/sml+json` for Procedure / SystemKind inline links. |
| **OpenAPI 3.1 / JSON Schema 2020-12** — optional-field semantics | "OpenAPI Specification 3.1.0" under *API Specification Standards* | Supporting: confirms absent optional fields are spec-conformant — establishes P4 (not P3). |

**Out-of-scope sources (not cited):** IANA Link Relations
registry (vocabulary-consistency hazard per eval §3); RFC 8288
(applies to supplementary `links[]`, not inline `@link`).

> **SensorML / `application/sml+json` reference-list verification:**
> drafter must confirm the SensorML JSON encoding entry exists in
> the curated references list. If absent, surface to user before
> recommending the `application/sml+json` constant in an upstream
> issue body. (Verification deferred to issue-filing step.)

## 5. Alternatives considered (internal-only)

### Option A (recommended) — Two-phase enrichment at the formatter projection site

- **Phase 1 — cheap pass.** At every inline-link
  `Link{Href: ...}` literal in the JSON / GeoJSON / SensorML
  formatters, add `Type: formaters.GeoJSONContentType` (or
  `SensorMLContentType` for procedure/systemKind). One-line edit
  per site. Zero DB cost. Independently shippable.
- **Phase 2 — enrichment pass.** Thread the existing
  `ResourceCache` (extending to Deployment / SamplingFeature /
  Observation / Command resource types as needed) into formatter
  projection sites. Populate `Title = <resource>.Name` and
  `UID = &<resource>.UniqueIdentifier` from the cache lookup at
  serialize time. `SerializeAll` paths preload the cache for the
  page; per-item `Serialize` falls back to direct lookup or skips
  enrichment.

**Why two phases:** they have independent acceptance criteria,
independent risk profiles (Phase 1 is constant-only; Phase 2 has
a batched-preload correctness obligation), and can ship as one
or two PRs.

### Option B (rejected) — populate `Rel` on inline `@link`

The eval flagged that the property name (`system@link`) already
conveys the relation, and the supplementary `links[]` already
carries the formal `ogc-rel:*` rel. Populating `Rel` on inline
`@link` adds duplication without information. **Reject; recommend
omitting `Rel` from inline `@link`.**

### Option C (rejected) — IANA `parent`

Introducing IANA-registered `parent` rel on inline `@link` would
fragment the upstream's existing `ogc-rel:*` vocabulary.
**Reject** per eval's vocabulary-consistency argument.

## 6. Recommended fix

**Option A (two-phase).** Acceptance criteria stated independently
per phase to permit single-PR or split-PR.

### Phase 1 — `Type` constants (cheap pass)

Edit every inline-link `Link{Href: ...}` literal at the formatter
projection sites enumerated by report-11's audit. Add the
appropriate constant per the §2.1 mapping table. Reuse existing
`GeoJSONContentType` / `SensorMLContentType` constants in
`association_links.go`.

**Acceptance:** unit test asserting non-empty `type` field on
inline `@link` for each of the 14 inline-link properties across
the 7 resource types in fixtures.

### Phase 2 — `Title` and `UID` (enrichment pass)

Extend `ResourceCache` to cover the additional linked-resource
types not already cached (Deployment, SamplingFeature,
Observation result references, Command procedure references —
modulo whatever the existing cache already covers via
`FetchParentSystems` / `FetchProcedures` extension precedent).
Thread the cache through the formatter `SerializeAll` paths;
populate `Title` + `UID` from cache lookups at projection time.

**Acceptance:**
1. unit test asserting populated `title` + `uid` on inline `@link`
   for each enrichable property in fixtures with non-nil linked
   resources;
2. integration test asserting a single page of N items issues
   **at most O(distinct-linked-resource-types) DB queries**, not
   O(N) (N+1 regression guard);
3. T2 round-trip-preservation regression-guard
   (issue-025 live-test §T2) still passes — client-supplied
   optionals on input round-trip preserved unchanged.

Implementation surface (rough): Phase 1 ≈ 10–15 one-line edits.
Phase 2 ≈ 1 ResourceCache extension + 1 thread-through per
formatter family + N+1 regression test.

## 7. Scope guard

What NOT to touch as part of this filing:

- Supplementary `links[]` enrichment — `Type`/`Title`/`UID`
  population there is already in place via `association_links.go`.
- Inline `@link.href` absolutization — separate filing
  ([`./report-11-inline-link-absolutization-remaining-5.md`](./report-11-inline-link-absolutization-remaining-5.md)).
- `Link` struct definition — already supports all five fields.
- `Rel` on inline `@link` — recommend omit (Option B); avoid IANA
  `parent` (Option C) per vocabulary consistency.
- Client-side round-trip behavior on input — T2 regression guard.
- Schema migration — wire-format change only; no DB changes.
- Type values for `result@link` (Observation) where MIME is
  determined by `resultEncoding` per record — defer to upstream's
  judgement; recommend omit `Type` rather than guess.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#25` (parent) was closed by
  `3fa1b0c`, `704a9e3`, and `d2d1347`. Maintainer self-acknowledged
  the residual on closure: *"Mostly there for most associations
  however not all fully enriched"*. This filing tracks that
  acknowledged residual to a discrete issue.
- Companion filing [`report-11`](./report-11-inline-link-absolutization-remaining-5.md)
  addresses the href-correctness axis on the same emission sites.
  Reviewers should see both filings sequenced.
- No fork-side patch; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Phasing — single PR or two? | **One issue describing two phases with independent acceptance criteria.** Maintainer can split. |
| `Rel` on inline `@link` — populate or omit? | **Omit** (Option A in plan §8 Q2). Property name conveys the relation; supplementary `links[]` carries the formal `ogc-rel:*`. |
| N+1 avoidance pattern | **Existing `ResourceCache` in `association_links.go`** is the precedent. Issue body cites it; line-level threading deferred to fix PR. |
| Coverage scope | **All 14 inline-link properties across 7 resource types.** No supplementary `links[]` scope-creep. |
| Severity P4 vs P3? | **P4 retained.** All enriched fields spec-optional; no conformance violation. |
| Live reproducer required? | **No.** Defect is structural (every projection site constructs `Link{Href: ...}` with zero-valued enrichment fields); T2 regression guard sufficient. |
| Cite eval/evidence paths in body? | **Yes** — validation-chain footer. |
| Plan's framing accuracy | **Plan was accurate on the inline-link gap; plan's framing of supplementary `links[]` "rel only" is outdated.** Supplementary `links[]` already populates Type/Title/UID via `ResourceCache`. Report adjusts the framing accordingly: residual is inline-only. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P4] Inline @link enrichment: populate type/title/uid on the formatter projection (residual from #25)`

**Labels:** `enhancement`

---

### Context

Issue #25 was closed by `3fa1b0c` + `704a9e3` (supplementary
`links[]` `Rel` population) and `d2d1347` (inline `@link`
reorganized as a wire-format projection carrying `Href` only).
On closure the maintainer acknowledged: *"Mostly there for most
associations however not all fully enriched"*. This issue tracks
that residual.

The supplementary `links[]` array is now richly enriched —
`internal/model/formaters/association_links.go` populates `Type`,
`Title`, and `UID` (with a `ResourceCache` to avoid N+1) on every
supplementary entry covered. Inline `@link` emission is
**`Href`-only** at every formatter projection site post-`d2d1347`.

### Claim

Three independent enrichment gaps on inline `@link` (all
spec-optional per OAS31 lines 312-372 — no conformance violation):

- **`type`** — constant per inline-link property (`application/geo+json`
  for System/Deployment/SamplingFeature; `application/sml+json` for
  Procedure/SystemKind). Cheap, no DB cost.
- **`title`** — linked resource's `Name`. Requires a lookup;
  pattern already exists in this repo via `ResourceCache`.
- **`uid`** — linked resource's `UniqueIdentifier`. Same lookup
  path as `title`; populated in tandem in
  `association_links.go:141-144 / 197-200 / 227-230 / 282-285`.

Coverage: 14 inline-link properties across 7 resource types
(Datastream, ControlStream, Command, Observation, Deployment,
SamplingFeature, System) — see report-11 for the full inline-link
inventory.

### Static evidence

`upstream/main` HEAD `df6da0d`:

```go
// internal/model/common_shared/links.go — struct supports all five fields:
type Link struct {
    Href  string  `json:"href"`
    Rel   string  `json:"rel,omitempty"`
    Type  string  `json:"type,omitempty"`
    Title string  `json:"title,omitempty"`
    UID   *string `json:"uid,omitempty"`
}
```

```go
// internal/model/formaters/json_formatters/datastream_json.go — Href-only projection:
out.SystemLink = &common_shared.Link{
    Href: formaters.ToFunctionalAssociationHref("/systems/" + *datastream.SystemID),
}
// (identical shape in control_stream_json.go and at every other
// inline-link projection site enumerated by the report-11 audit.)
```

```text
// MIME-type constants already defined in this repo:
internal/model/formaters/association_links.go:15: GeoJSONContentType  = "application/geo+json"
internal/model/formaters/association_links.go:16: SensorMLContentType = "application/sml+json"
```

### Live evidence

Not included — defect is structural (every inline-link projection
constructs `common_shared.Link{Href: ...}` with zero-valued
enrichment fields, omitted under `,omitempty`). Parent #25's
pre-fix matrix demonstrated the same wire-format gap; the
post-`d2d1347` reorganization moved synthesis to the formatter
without altering the field-set populated.

### Recommended fix — two phases with independent acceptance criteria

**Phase 1 — `type` constants (cheap pass).** At every inline-link
`Link{Href: ...}` literal, add the appropriate MIME constant
already defined in `association_links.go`:

```go
out.SystemLink = &common_shared.Link{
    Href: formaters.ToFunctionalAssociationHref("/systems/" + *datastream.SystemID),
    Type: formaters.GeoJSONContentType,
}
```

Mapping:

| Inline-link property | Recommended `Type` |
|---|---|
| `system@link`, `parentSystem@link`, `subsystems@link`, `deployedSystems@link`, `deployment@link`, `parentDeployment@link`, `subdeployments@link`, `featureOfInterest@link`, `samplingFeature@link`, `sampledFeature@link`, `platform@link` | `application/geo+json` |
| `procedure@link`, `systemKind@link` | `application/sml+json` |
| `result@link` (Observation) | omit (MIME varies by `resultEncoding`) |

Acceptance: unit test asserting non-empty `type` on inline `@link`
in fixtures of all 7 resource types.

**Phase 2 — `title` and `uid` (enrichment pass).** Extend the
existing `ResourceCache` (used by `association_links.go` for
supplementary `links[]` enrichment) to additional linked-resource
types as needed; thread it into the formatter `SerializeAll`
paths; populate `title = <resource>.Name` +
`uid = &<resource>.UniqueIdentifier`.

Acceptance:

1. Unit test asserting populated `title` + `uid` on inline `@link`
   in fixtures with non-nil linked resources.
2. Integration test asserting a single page of N items issues at
   most O(distinct-linked-resource-types) DB queries — N+1
   regression guard.
3. Round-trip preservation of client-supplied optional fields on
   POST→GET remains unchanged (regression guard from #25's
   pre-fix matrix T2).

### `Rel` on inline `@link`?

Recommend **omit**. The property name (`system@link`) already
conveys the relation, and the supplementary `links[]` carries the
formal `ogc-rel:*` rel via `OGCRel(...)`. Populating `Rel` on
inline `@link` would duplicate without adding information.
**Do not introduce IANA `parent`** — fragment the existing
`ogc-rel:*` vocabulary.

### Spec authority

- **OGC 23-001 / 23-002** CSAPI Part 1 + Part 2 OAS31 — Link
  object (OAS31 lines 312-372): `href` required; `rel`, `type`,
  `title`, `uid` all optional. Authority for "purely additive,
  no client breakage."
- **RFC 7946** — `application/geo+json`.
- **OGC SensorML JSON Encoding** — `application/sml+json`.
- **OpenAPI 3.1 / JSON Schema 2020-12** — absent optional fields
  are spec-conformant; establishes P4 not P3.

### Severity

**P4** — UX, no spec violation. All enriched fields spec-optional
with no required-when-known clause. Non-population is conformant
but loss-of-information for clients.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-025.md`](../issue-evaluations/issue-025.md) §"Recommended fix scope"
- Evidence (static): [`docs/research/evidence/issue-025/static-analysis-2026-04-30.md`](../evidence/issue-025/static-analysis-2026-04-30.md)
- Evidence (live, parent matrix): [`docs/research/evidence/issue-025/live-test-2026-04-30.md`](../evidence/issue-025/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-025/spec-authority-2026-04-30.md`](../evidence/issue-025/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-12-inline-link-type-title-uid-enrichment.md`](../upstream-issues/plan-12-inline-link-type-title-uid-enrichment.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #16
- Companion filing (href-correctness axis on same emission sites): [`docs/research/upstream-issue-reports/report-11-inline-link-absolutization-remaining-5.md`](./report-11-inline-link-absolutization-remaining-5.md)
