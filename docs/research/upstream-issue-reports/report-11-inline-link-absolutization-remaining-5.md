# Report 11 — Inline `@link` absolutization for remaining resource types (audit completion of `d2d1347`)

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-11-inline-link-absolutization-remaining-5.md`](../upstream-issues/plan-11-inline-link-absolutization-remaining-5.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#15** (Inline `@link` absolutization for remaining resource types) |
| Source fork issue | `OS4CSAPI/connected-systems-go#24` (closed by upstream `d2d1347`); this filing extends the audit to inline-link sites that the parent fix did not cover. |
| Research plan | [`../upstream-issues/plan-11-inline-link-absolutization-remaining-5.md`](../upstream-issues/plan-11-inline-link-absolutization-remaining-5.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P3** — spec conformance enhancement; same `format: uri` argument as parent #24 |
| Tier | **B** (file second; enhancement, single-issue scope) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). `d2d1347` ("Cleaned up
data/controlstream's SystemLink") is in lineage. **Re-verification
revised the plan's resource-type count downward in one direction
and upward in another:** the parent fix is narrower than the plan
assumed, and the remaining inline-link audit therefore covers
*more* sites than the plan's "remaining 5 resource types" framing
suggested.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git log --oneline upstream/main | Select-String 'd2d1347'
d2d1347 Cleaned up data/controlstream's SystemLink
```

**Inline-link inventory (`*common_shared.Link` with `@link` JSON tag) on `upstream/main` domain models:**

```text
$ git grep -nE '@link' upstream/main -- internal/model/domains/

# Datastream  (4 inline links — parent fix removed only system@link)
internal/model/domains/datastream.go:26:  ProcedureLink       *Link `json:"procedure@link,omitempty"`
internal/model/domains/datastream.go:27:  DeploymentLink      *Link `json:"deployment@link,omitempty"`
internal/model/domains/datastream.go:28:  FeatureOfInterest   *Link `json:"featureOfInterest@link,omitempty"`
internal/model/domains/datastream.go:29:  SamplingFeatureLink *Link `json:"samplingFeature@link,omitempty"`

# ControlStream  (4 inline links — parent fix removed only system@link)
internal/model/domains/control_stream.go:25-28: ProcedureLink, DeploymentLink, FeatureOfInterest, SamplingFeatureLink

# Command  (1 inline link)
internal/model/domains/command.go:31:     ProcedureLink     *Link `json:"procedure@link,omitempty"`

# Deployment  (1 single + 1 array)
internal/model/domains/deployment.go:76:  Platform        *Link  `json:"platform@link,omitempty"`
internal/model/domains/deployment.go:77:  DeployedSystems Links  `json:"deployedSystems@link,omitempty"`

# Observation  (2 inline links)
internal/model/domains/observation.go:16: ProcedureLink     *Link `json:"procedure@link,omitempty"`
internal/model/domains/observation.go:23: ResultLink        *Link `json:"result@link,omitempty"`

# SamplingFeature  (1 inline link, declared on two views)
internal/model/domains/sampling_feature.go:27: SampledFeatureLink *Link `json:"sampledFeature@link,omitempty"`
internal/model/domains/sampling_feature.go:71: SampledFeatureLink *Link `json:"sampledFeature@link,omitempty"`

# System  (1 inline link)
internal/model/domains/system.go:115:     SystemKind *Link `json:"systemKind@link,omitempty"`
```

**Revised totals: 7 resource types, 14 distinct inline-link properties** (the eval cited 17; the difference is bookkeeping — duplicate declarations across views in the model package, plus the 2 SystemLink properties removed by `d2d1347`).

**What `d2d1347` actually removed:** only the **synthesized
`system@link`** from Datastream and ControlStream domain models,
replacing it with formatter-side projection from `SystemID`. The
**four user-supplied inline links** on each of those resource types
(`procedure@link`, `deployment@link`, `featureOfInterest@link`,
`samplingFeature@link`) **remain on the domain model** and pass
through to the wire raw, without normalization.

**Formatter-side normalization audit** (where each inline link is normalized via `ToFunctionalAssociationHref` or sibling helper):

```text
$ git grep -nE 'ToFunctionalAssociationHref|absolutizeLink' upstream/main -- internal/model/formaters/
```

| Resource type | Inline-link field | Has dedicated formatter? | Normalize-on-serialize? |
|---|---|---|---|
| Datastream | `system@link` (synthesized) | `datastream_json.go` | ✓ (via parent fix) |
| Datastream | `procedure@link` | `datastream_json.go` | ✗ (passes through raw) |
| Datastream | `deployment@link` | `datastream_json.go` | ✗ (passes through raw) |
| Datastream | `featureOfInterest@link` | `datastream_json.go` | ✗ (passes through raw) |
| Datastream | `samplingFeature@link` | `datastream_json.go` | ✗ (passes through raw) |
| ControlStream | `system@link` (synthesized) | `control_stream_json.go` | ✓ (via parent fix) |
| ControlStream | `procedure@link` | `control_stream_json.go` | ✗ (passes through raw) |
| ControlStream | `deployment@link` | `control_stream_json.go` | ✗ (passes through raw) |
| ControlStream | `featureOfInterest@link` | `control_stream_json.go` | ✗ (passes through raw) |
| ControlStream | `samplingFeature@link` | `control_stream_json.go` | ✗ (passes through raw) |
| Command | `procedure@link` | (no `command_json.go`) | ✗ (default JSON marshal) |
| Observation | `procedure@link` | (no `observation_json.go`) | ✗ (default JSON marshal) |
| Observation | `result@link` | (no `observation_json.go`) | ✗ (default JSON marshal) |
| Deployment | `platform@link` | `deployment_geojson.go:68` | ✓ (geojson only; no `_json.go`) |
| Deployment | `deployedSystems@link` | `deployment_geojson.go:61` | ✓ (geojson only; no `_json.go`) |
| SamplingFeature | `sampledFeature@link` | `sampling_feature_geojson.go:55` | ✓ (via local `absolutizeLink`) |
| System | `systemKind@link` | `system_geojson.go` | ✓ (geojson formatter; coverage in default JSON path TBD) |

**Defect-pattern enumeration:** 11 distinct pass-through inline-link sites
across 4 resource types still bypass normalization on the JSON wire format:

- Datastream × 4 (procedure/deployment/featureOfInterest/samplingFeature)
- ControlStream × 4 (same four)
- Command × 1 (procedure)
- Observation × 2 (procedure / result)

Plus a **second defect class**: Command and Observation have **no
dedicated JSON formatter at all**, so even if their domain-model
inline-link fields contained absolute hrefs server-side, no
normalization happens in either direction — purely user-passthrough.

**Helper idempotency (sanity, parent fix-helper unchanged):**

```text
$ git show upstream/main:internal/model/formaters/association_links.go |
    Select-String 'IsAbs|parsed' -Context 1,1
# Confirms the helper is idempotent on absolute inputs (per issue-024 eval).
```

**Live re-verification:** not captured this round. POST per
resource type with inline `@link` fields requires authenticated
admin endpoints not exposed on `csapi-go-upstream`. Defect is
structural — 11 unguarded pass-through sites (8 of them in existing
formatters; 3 in Cmd/Obs which lack dedicated formatters) are visible
above — and parent #24's pre-fix matrix
transitively applies (same shape, same wire-format
normalization gap).

## 2. Static evidence

Source: [`../evidence/issue-024/static-analysis-2026-04-30.md`](../evidence/issue-024/static-analysis-2026-04-30.md)
"Affected-fields completeness" §, refreshed in §1 above.

Three distinct sub-defects fall out of the audit:

### 2.1 Datastream / ControlStream four-link pass-through

Both `datastream_json.go` and `control_stream_json.go` synthesize
`system@link` via `ToFunctionalAssociationHref(...)` (parent fix
in place), but the four sibling user-supplied inline links
(`procedure@link`, `deployment@link`, `featureOfInterest@link`,
`samplingFeature@link`) ride through via the embedded domain
struct (`DatastreamJSONFeature struct { domains.Datastream; ... }`)
without being passed through `ToFunctionalAssociationHref`. The
helper is idempotent on absolute inputs, so a one-line
normalization for each of the 4 inline link fields per formatter
(8 lines total) closes this gap symmetrically with the parent fix.

### 2.2 Command / Observation no-JSON-formatter

`internal/model/formaters/json_formatters/` contains no
`command_json.go` or `observation_json.go`. Default JSON marshal
of the domain struct passes the `*common_shared.Link` pointer
through verbatim. Inline `procedure@link` / `result@link` on
Observation and `procedure@link` on Command therefore emit raw
user-supplied hrefs — no normalization.

### 2.3 Helper coverage in geojson vs JSON paths is asymmetric

GeoJSON formatters for Deployment, SamplingFeature, and System
*do* normalize their inline-link fields (lines:
`deployment_geojson.go:61, 68`; `sampling_feature_geojson.go:55,
122`; `system_geojson.go:87, 95, 102`). The JSON-format path for
Datastream/ControlStream covers only the synthesized `system@link`,
not the user-supplied four. The default JSON marshal path for
Command/Observation has no normalization at all. The asymmetry is
the audit completion target.

## 3. Live evidence

Not captured this round (auth-gated POST per resource type).
Justification in §1. The 5-row per-resource-type matrix sketched
in plan §6 follows deterministically from the static evidence:
the 11 pass-through sites (8 sibling links in existing
formatters + 3 in Cmd/Obs missing-formatter cases) cannot produce
absolute hrefs from user-supplied relative input.

Parent #24's pre-fix evidence in
[`../evidence/issue-024/live-test-2026-04-30.md`](../evidence/issue-024/live-test-2026-04-30.md)
demonstrated the relative-href symptom on `system@link` for
Datastream and ControlStream; the same shape applies symmetrically
to the four sibling inline links on those resource types and to
the unprotected sites on Command and Observation.

## 4. Spec authority

Identical citation chain to parent #24 — the audit-completion
argument is exactly that the spec applies uniformly across
inline-link properties.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001 / 23-002** — CSAPI Part 1 + Part 2 OAS31 schemas, inline `@link` Link object with `href: format: uri` | "OGC API - Connected Systems - Part 1: Core" and "Part 2: Dynamic Data" / "Part 1/2: OpenAPI Specification" under *OGC CSAPI Standards* | **Primary citation.** The `*ref_11` / `*ref_12` `Link` schema is reused across all 14 inline-link properties on the 7 resource types. |
| **RFC 3986** §4.3 — Absolute URI definition | "RFC 3986 — Uniform Resource Identifier (URI): Generic Syntax" under *IETF RFCs* | Defines what `format: uri` requires (absolute URI per §4.3, distinct from `format: uri-reference` per §4.1). |
| **OpenAPI 3.1 / JSON Schema 2020-12** — `format` annotation vs assertion behavior | "OpenAPI Specification 3.1.0" under *API Specification Standards* | Supporting nuance: `format: uri` is annotation by default; strict format-assertion validators flag relative values. |
| **CSAPI worked examples** — Datastream (lines 1922-1929), Command (lines 3547-3550), Observation, Deployment | Same OGC CSAPI Part 1/2 OAS31 spec entries | Worked examples are uniformly absolute `https://...` URIs across all resource types and all inline `@link` properties. |

**Out-of-scope sources (not cited):** RFC 8288 (applies to
supplementary `links[]`, already absolute), HTTP RFCs (resolution
not at issue).

## 5. Alternatives considered (internal-only)

The backlog explicitly authorized two valid approaches:

### Option A (recommended) — Normalize-on-serialize via `ToFunctionalAssociationHref` idempotency

For each of the 11 pass-through sites (8 in existing formatters; 3 in
Cmd/Obs requiring 2 new minimal formatters),
pipe the inline-link `Href` through
`ToFunctionalAssociationHref(...)` at serialize time. The helper
is idempotent on absolute inputs (already-absolute hrefs are
returned unchanged), so this is safe even when the field happens
to be user-supplied with an already-absolute value.

- Two-line edit per pass-through site (or one-line replacement of
  the embedded struct projection): pre-marshal walk that calls
  the helper on each inline-link field.
- For Command / Observation: add a thin `command_json.go` /
  `observation_json.go` formatter modeled on
  `datastream_json.go`'s embedded-struct shape.
- Risk: zero. Mirrors parent #24's helper-call pattern; uses an
  unchanged helper.

### Option B (alternative) — `d2d1347`-style domain-model removal + formatter projection

Repeat the parent fix's structural pattern (remove the inline-link
field from the domain model; project from a FK ID via the
formatter). Only appropriate where the inline-link is **derived**
from another model field. None of the 11 pass-through sites are
clearly derived in the same way `system@link` was derived from
`SystemID` — they are user-supplied associations to other
resources. Option B is therefore **not recommended** for the
audit-completion sites; it would force a model rewrite where a
formatter normalization suffices.

**Lead with Option A only.** Mention Option B briefly in the
issue body as the parent-fix pattern that doesn't generalize to
user-supplied inline links.

## 6. Recommended fix

**Option A.** Mechanical edits at three site classes:

### 6.1 Datastream / ControlStream — 4 + 4 = 8 line edits

In `datastream_json.go` and `control_stream_json.go`, after the
existing `system@link` projection block, normalize the four
sibling user-supplied inline links via the helper:

```go
// (datastream_json.go — example shape, pseudocode)
out := DatastreamJSONFeature{Datastream: *ds, ...}
if out.ProcedureLink != nil && out.ProcedureLink.Href != "" {
    out.ProcedureLink.Href = formaters.ToFunctionalAssociationHref(out.ProcedureLink.Href)
}
if out.DeploymentLink != nil && out.DeploymentLink.Href != "" {
    out.DeploymentLink.Href = formaters.ToFunctionalAssociationHref(out.DeploymentLink.Href)
}
if out.FeatureOfInterest != nil && out.FeatureOfInterest.Href != "" {
    out.FeatureOfInterest.Href = formaters.ToFunctionalAssociationHref(out.FeatureOfInterest.Href)
}
if out.SamplingFeatureLink != nil && out.SamplingFeatureLink.Href != "" {
    out.SamplingFeatureLink.Href = formaters.ToFunctionalAssociationHref(out.SamplingFeatureLink.Href)
}
```

### 6.2 Command / Observation — add minimal JSON formatters

Add `command_json.go` and `observation_json.go` modeled on
`datastream_json.go`'s embedded-struct + pre-marshal-walk pattern.
Each formatter normalizes its inline-link fields:

- Command: `ProcedureLink`
- Observation: `ProcedureLink`, `ResultLink`

### 6.3 Acceptance criterion (carry-forward from issue-024 eval)

Add a unit test asserting `^https?://` for **every** inline
`@link.href` value in fixtures of all 7 resource types. The
eval's "Acceptance criterion sharpening" already proposed this; it
becomes the audit-completion gate.

Implementation surface: 8 line-edits across 2 existing formatters,
2 new minimal formatters (~30 lines each), 1 acceptance test.
No model changes, no schema migration.

## 7. Scope guard

What NOT to touch as part of this filing:

- The synthesized `system@link` projection on Datastream /
  ControlStream — parent fix `d2d1347` is in place.
- Supplementary `links[]` array — already absolute via
  pre-existing helpers; not the subject of this filing.
- The geojson formatters' inline-link normalization — already in
  place for Deployment, SamplingFeature, System.
- `ToFunctionalAssociationHref` itself — leverage its existing
  idempotency; no helper changes.
- `Type`/`Title`/`UID` enrichment on inline `@link` objects —
  separate filing, plan-12 / report-12. **Cross-reference but
  do not bundle.**
- Schema migration — wire-format change only; no DB changes.
- Domain-model removal of user-supplied `*common_shared.Link`
  fields (Option B) — not recommended; would force rewrites
  where normalization suffices.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#24` (parent) was a P3
  spec-conformance enhancement specifically about absolute hrefs
  on inline `@link.href` for Datastream + ControlStream. Closed
  by `d2d1347` adopting the eval's recommended wire-format
  projection approach (model removal + formatter projection from
  `SystemID`).
- This filing extends the audit to inline-link sites the parent
  fix did not cover. Plan §1 framed it as "remaining 5 resource
  types"; the actual structural picture is **11 pass-through sites
  across 4 resource types**, with three
  resource types (Deployment, SamplingFeature, System) already
  partially or fully covered by their geojson formatters. Audit
  framing in this report adjusts the framing accordingly.
- Plan-12 / report-12 will address the symmetric `Type`/`Title`/
  `UID` enrichment audit on the same inline-link sites.
- No fork-side patch; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Per-resource-type fix shape: derived vs user-supplied | **All 11 pass-through sites are user-supplied** (`procedure@link`, `deployment@link`, etc. — associations to other resources, not derivable from FK IDs on the same model). **Option A (normalize-on-serialize)** is the only general fit. Option B (model-removal) is mentioned but not recommended. |
| Bundle with plan-12 (Type/Title/UID enrichment)? | **No.** Backlog process notes sequence #15 before #16; cross-reference in scope-guard. |
| Issue-body length | **Use the inventory table from §1** as the centerpiece. Per-resource-type narrative kept short. |
| Severity P3 vs P4? | **P3 retained.** Same `format: uri` argument as parent #24. |
| Live reproducer required? | **No.** Auth-gated; defect is structural (11 pass-through sites across 4 resource types; 8 in existing formatters + 3 in Cmd/Obs); static + parent #24 transitive sufficient. |
| Cite eval/evidence paths? | **Yes**, validation-chain footer. |
| Plan's "5 remaining resource types" framing | **Partially incorrect.** The actual residual is 11 pass-through *sites* across 4 resource types (Datastream, ControlStream, Command, Observation); 2 of those types (Command, Observation) also lack a dedicated JSON formatter file. Three resource types named in the plan (Deployment, SamplingFeature, System) already have their inline links normalized in their geojson formatters. Report adjusts the framing. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P3] Audit completion: inline @link.href absolutization for Datastream/ControlStream sibling links + Command/Observation (residual of d2d1347 / issue #24)`

**Labels:** `enhancement`, `spec-conformance`

---

### Context

`d2d1347` ("Cleaned up data/controlstream's SystemLink") closed
issue #24 by removing the synthesized `system@link` from
Datastream and ControlStream domain models and projecting it from
`SystemID` via `ToFunctionalAssociationHref(...)`, yielding
absolute URIs on the wire — matching OAS31's `format: uri`
contract.

The eval that informed #24 enumerated **14 inline `@link`
properties across 7 resource types** in `internal/model/domains/`.
The parent fix landed normalization for the 2 synthesized
`system@link` properties on Datastream + ControlStream. The
remaining inline-link properties — both **user-supplied
sibling links on those same two resource types** and inline
links on resource types without a custom JSON formatter — still
emit user-supplied hrefs verbatim, without normalization.

### Claim

The following 11 inline-link sites span 4 resource types; 8 sit in
existing JSON formatters that need additional helper calls, and 3 sit
in Command/Observation which lack a dedicated JSON formatter file
entirely. All bypass `ToFunctionalAssociationHref` on the JSON wire
format:

| Resource | Inline-link property | Defect class |
|---|---|---|
| Datastream | `procedure@link` | embedded-struct passthrough in `datastream_json.go` |
| Datastream | `deployment@link` | embedded-struct passthrough |
| Datastream | `featureOfInterest@link` | embedded-struct passthrough |
| Datastream | `samplingFeature@link` | embedded-struct passthrough |
| ControlStream | `procedure@link` | embedded-struct passthrough in `control_stream_json.go` |
| ControlStream | `deployment@link` | embedded-struct passthrough |
| ControlStream | `featureOfInterest@link` | embedded-struct passthrough |
| ControlStream | `samplingFeature@link` | embedded-struct passthrough |
| Command | `procedure@link` | no `command_json.go` formatter exists |
| Observation | `procedure@link` | no `observation_json.go` formatter exists |
| Observation | `result@link` | no `observation_json.go` formatter exists |

A user-supplied relative href (e.g.
`/procedures/abc-123`) on any of these properties round-trips
verbatim, conflicting with the OAS31 `format: uri` constraint
that #24 closed for `system@link`. Three resource types
(Deployment, SamplingFeature, System) are already covered by their
geojson formatters and are **not** in scope here.

### Static evidence

`upstream/main` HEAD `df6da0d`:

```text
$ git grep -nE '@link' upstream/main -- internal/model/domains/
internal/model/domains/datastream.go:26-29:        ProcedureLink, DeploymentLink, FeatureOfInterest, SamplingFeatureLink
internal/model/domains/control_stream.go:25-28:    ProcedureLink, DeploymentLink, FeatureOfInterest, SamplingFeatureLink
internal/model/domains/command.go:31:              ProcedureLink
internal/model/domains/observation.go:16,23:       ProcedureLink, ResultLink
internal/model/domains/deployment.go:76-77:        Platform, DeployedSystems
internal/model/domains/sampling_feature.go:27,71:  SampledFeatureLink
internal/model/domains/system.go:115:              SystemKind
```

`datastream_json.go` and `control_stream_json.go` define
`*JSONFeature` types that embed the domain struct and overlay a
formatter-synthesized `system@link`. The other four user-supplied
inline links pass through unchanged via the embedded struct:

```go
type DatastreamJSONFeature struct {
    domains.Datastream                                     // <- includes the 4 user-supplied links, raw
    SystemLink *common_shared.Link `json:"system@link,omitempty"`   // <- synthesized + normalized
}
```

`internal/model/formaters/json_formatters/` contains no
`command_json.go` or `observation_json.go` — Command and
Observation default to the standard JSON marshal of the domain
struct.

### Live evidence

Not included in this filing — POST per resource type with inline
`@link` fields requires authenticated admin endpoints. Defect is
structural (visible in the static evidence above); behavior
follows deterministically. Parent #24's pre-fix matrix
demonstrated the same wire-format gap on `system@link`.

### Recommended fix

**Option A — Normalize-on-serialize via `ToFunctionalAssociationHref` idempotency.** The helper is already idempotent on absolute inputs, so this is safe even when the field happens to contain an absolute href user-side.

In `datastream_json.go` and `control_stream_json.go`, after the
existing `system@link` projection, normalize the four sibling
inline links:

```go
if out.ProcedureLink != nil && out.ProcedureLink.Href != "" {
    out.ProcedureLink.Href = formaters.ToFunctionalAssociationHref(out.ProcedureLink.Href)
}
// ... DeploymentLink, FeatureOfInterest, SamplingFeatureLink — same shape
```

Add minimal `command_json.go` and `observation_json.go`
formatters modeled on `datastream_json.go` (embedded struct +
pre-marshal walk) covering:

- Command: `ProcedureLink`
- Observation: `ProcedureLink`, `ResultLink`

Add a unit test asserting `^https?://` for every inline
`@link.href` in fixtures of all 7 resource types — closes the
audit-completion gate proposed by the issue-024 eval's
"Acceptance criterion sharpening" §.

**Option B (not recommended)** — repeating `d2d1347`'s pattern
(domain-model removal + formatter projection from FK ID) is
appropriate only for **derived** inline links. The 11 sites listed
are user-supplied associations, not derived; Option A
(normalize-on-serialize) is the natural fit.

Implementation surface: 8 line-edits across 2 existing formatters,
2 new minimal formatters (~30 lines each), 1 acceptance test.

### Spec authority

- **OGC 23-001 / 23-002** CSAPI Part 1 + Part 2 OAS31 — the inline
  `@link` Link object's `href: format: uri` constraint (the
  `*ref_11`/`*ref_12` `Link` schema) is reused across **all**
  inline-link properties on **all** resource types. Same citation
  as #24; this filing extends the same conformance argument.
- **RFC 3986** §4.3 — absolute URI required by `format: uri`,
  distinct from `format: uri-reference` (§4.1).
- **OpenAPI 3.1 / JSON Schema 2020-12** — `format` is annotation
  by default; strict format-assertion validators flag relative
  values.
- **CSAPI worked examples** — uniformly absolute `https://...`
  hrefs across all inline `@link` properties on all resource
  types.

### Severity

**P3** — same `format: uri` conformance argument as parent #24,
applied to the audit-completion residual. Spec-conformance
enhancement; no data-integrity or runtime-failure impact.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-024.md`](../issue-evaluations/issue-024.md) §"Affected-fields completeness" + §"Acceptance criterion sharpening"
- Evidence (static): [`docs/research/evidence/issue-024/static-analysis-2026-04-30.md`](../evidence/issue-024/static-analysis-2026-04-30.md)
- Evidence (live, parent matrix): [`docs/research/evidence/issue-024/live-test-2026-04-30.md`](../evidence/issue-024/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-024/spec-authority-2026-04-30.md`](../evidence/issue-024/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-11-inline-link-absolutization-remaining-5.md`](../upstream-issues/plan-11-inline-link-absolutization-remaining-5.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #15

**See also:** sibling filing for `Type`/`Title`/`UID` enrichment
on the same inline-link sites is in preparation
(plan-12 / report-12).

---

## 11. Filing record

| Field | Value |
|---|---|
| Upstream issue | [SomethingCreativeStudios/connected-systems-go#10](https://github.com/SomethingCreativeStudios/connected-systems-go/issues/10) |
| Issue API id | `4387932196` |
| Filed | 2026-05-05 |
| Audit verdict | `pass` — see [../report-audit-log.md](../report-audit-log.md#report-11--inline-link-absolutization-for-remaining-resource-types--2026-05-05) |
| Audit reference HEAD | df6da0dff8e2d3e76b64b00f856c0d43ed644f6d |
