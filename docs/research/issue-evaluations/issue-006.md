# Issue #6 — Cross-resource references: `@link` objects vs flat `@id` strings

**Issue URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/6
**Type:** Research spike (P4-Informational)
**Verdict:** **Research-resolved. No code changes recommended on Datastream/ControlStream `@link` shape. Two minor conformance/UX gaps surfaced as candidate follow-ups.**

---

## 1. The premise — and why it is inverted

The issue states:

| Server         | `system@id` (flat) | `system@link` (object) |
|----------------|--------------------|------------------------|
| OSH SensorHub  | ✅ Present         | ❌ Absent              |
| cs-go          | ❌ Absent          | ✅ Present             |

…and asks whether cs-go should also emit `system@id` for backward compat.

After spec inspection, this framing is the wrong way around. **The
spec-conformant shape is `system@link` (object) on Datastream/ControlStream;
`system@id` is not defined for those resources at all.** SensorHub's emission
is the non-conformant variant.

## 2. What OGC 23-002 actually mandates

A line-by-line inventory of `.tmp-csapi-part2.yaml` (full table in the
evidence file `static-analysis-2026-04-30.txt`) yields a clear rule:

- **`@link` (object with required `href`)** is used wherever the reference is
  hierarchical, cross-collection, or carries metadata clients may need without
  another fetch. The complete inventory at the resource level:
  - Datastream: `system@link`, `procedure@link`, `deployment@link`,
    `featureOfInterest@link`, `samplingFeature@link`
  - ControlStream: same five fields
  - Observation: `procedure@link`, `result@link`
  - Command: `procedure@link`
  - CommandStatus: `observation@link` / `observationSet@link` /
    `datastream@link` / `external@link` (`oneOf`)
  - System: `systemKind@link`
  - SamplingFeature: `sampledFeature@link`
  - Deployment: `platform@link`, `deployedSystems@link`

- **`@id` (flat string)** is used only for the immediate parent of a
  sub-resource where the parent collection is fixed by context, plus a few
  same-collection sibling refs:
  - Observation: `datastream@id` (REQUIRED), `samplingFeature@id`
  - Command: `controlstream@id` (REQUIRED), `samplingFeature@id`
  - CommandStatus: `command@id` (REQUIRED)

There is **no `system@id` field** defined on Datastream or ControlStream
anywhere in the spec. The shape the issue hopes for does not exist.

The implicit design rule is: `@link` when the target resource lives in a
different collection or you may want a typed, titled, optionally
content-typed reference; `@id` when the target collection is implied by the
referencing resource's own context (an observation always references a
datastream, by definition).

## 3. cs-go's implementation — checked field by field

`internal/model/domains/*.go` mirrors the spec exactly:

| Resource model       | Field tag                              | Spec match |
|---------------------|----------------------------------------|------------|
| `Datastream.SystemLink`        | `json:"system@link,omitempty"`         | ✓ |
| `Datastream.ProcedureLink`     | `json:"procedure@link,omitempty"`      | ✓ |
| `Datastream.DeploymentLink`    | `json:"deployment@link,omitempty"`     | ✓ |
| `Datastream.FeatureOfInterest` | `json:"featureOfInterest@link,...`     | ✓ |
| `Datastream.SamplingFeatureLink`| `json:"samplingFeature@link,...`      | ✓ |
| `ControlStream.*Link`          | (same five)                            | ✓ |
| `Observation.DatastreamID`     | `json:"datastream@id"` (no omitempty)  | ✓ (required by spec) |
| `Observation.SamplingFeatureID`| `json:"samplingFeature@id,omitempty"`  | ✓ |
| `Observation.ProcedureLink`    | `json:"procedure@link,omitempty"`      | ✓ |
| `Command.ControlStreamID`      | `json:"controlstream@id"`              | ✓ |
| `Command.SamplingFeatureID`    | `json:"samplingFeature@id,omitempty"`  | ✓ |
| `Deployment.Platform`/`DeployedSystems` | `platform@link` / `deployedSystems@link` | ✓ |
| `SamplingFeature.SampledFeatureLink` | `sampledFeature@link`             | ✓ |
| `System.SystemKind`            | `systemKind@link`                      | ✓ |

Same fork (`SomethingCreativeStudios/connected-systems-go`) ships identical
struct tags, so this is upstream-stable.

## 4. Live confirmation on HEAD (4b994212)

**T1 — GET /datastreams?limit=1** (full transcript:
`docs/research/evidence/issue-006/live-tests-head-2026-04-30.txt`):

```json
{
  "id": "cf4cd80c-...",
  "name": "DS without uid A",
  "system@link": { "href": "systems/666ed3fe-..." },
  "outputName": "temperature",
  "links": [
    { "href": "https://.../systems/666ed3fe-...",
      "rel":  "ogc-rel:systems" },
    { "href": "https://.../datastreams/cf4cd80c-.../observations",
      "rel":  "ogc-rel:observations" }
  ],
  "Systems": null
}
```

**T3 — GET /observations?limit=1**:

```json
{
  "id": "17a6829b-...",
  "datastream@id": "b93d0040-...",
  "phenomenonTime": "2026-04-30T17:00:00Z",
  "resultTime":     "2026-04-30T17:00:00Z",
  "result": 42
}
```

Both shapes are spec-conformant. Clients have **two** discoverability paths
to the parent system from a datastream:

1. `system@link.href` → relative "systems/<uuid>" (spec-mandated)
2. `links[ rel="ogc-rel:systems" ].href` → absolute URL (derived
   association link added by `appendDatastreamAssociationLinks` in
   `json_formatters/datastream_json.go:62-86`)

## 5. Verdicts on the issue's research questions

**Q1. What does OGC 23-002 specify?**
`@link` for the cross-collection / hierarchical refs listed in §2.
`@id` for in-collection sibling refs and observation/command parent.
Both formats coexist by design; they are **not** alternative encodings of the
same reference.

**Q2. Are `@id` and `@link` alternative encodings of the same relationship?**
**No.** They are used at different positions in the data model and carry
different semantics (`@link` can carry rel/type/title/uid; `@id` is a bare
local-ID lookup whose target collection is contextually fixed).

**Q3. Should a conformant server emit BOTH for a given relationship?**
**No.** The spec defines exactly one shape per relationship. Adding
`system@id` alongside `system@link` on Datastream/ControlStream would
introduce a non-spec extension property with two sources of truth (drift
risk, validator failures against strict OAS31). **Recommendation: do not
emit `system@id` on Datastream or ControlStream.** The interop fix belongs
on the client side (already done in [ogc-client-CSAPI_2 #166](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/issues/166))
or on SensorHub (file an issue against OSH if cross-stack interop is a
project priority).

**Q4. Should `@link.href` be relative or absolute?**
The spec declares `format: uri` (RFC 3986; typically absolute) with an
absolute example. cs-go currently emits `"systems/<uuid>"` (relative
reference). Both are usable by clients given the request base URI, but
strict OAS31 validators may flag the relative form. **Candidate P3
follow-up: emit absolute URIs in inline `@link.href` properties** to match
the example and remove the validator surface; the supplementary `links[]`
array already does this.

**Q5. How do other OGC API implementations handle this?**
Out-of-scope for this spike (no installation of pygeoapi/ldproxy on hand).
Spec authority is sufficient: `@link` is the conformant shape and any other
shape is a fork-specific deviation.

**Q6. Should cs-go also emit `system@id` for backward compatibility?**
**No.** See Q3.

**Q7. Other cross-reference patterns?**
Inventoried in §2. The full list emitted by cs-go: `procedure@link`,
`deployment@link`, `featureOfInterest@link`, `samplingFeature@link`,
`platform@link`, `deployedSystems@link`, `sampledFeature@link`,
`systemKind@link`, `result@link`, plus the `@id` parents on observation /
command / commandStatus.

## 6. Conformance/UX gaps surfaced (candidate follow-ups)

These are not Issue #6 itself, but were turned up by the deep evaluation:

- **G1 — P3 [enhancement] `@link.href` relative vs absolute.**
  Inline `system@link.href` is `"systems/<uuid>"`. Spec example is absolute.
  Either emit absolute (preferred — matches example, friendlier to strict
  OAS validators, matches behavior of the supplementary `links[]` array) or
  document the resolution rule. Trivial fix in
  `json_formatters/datastream_json.go` and the equivalent control-stream
  formatter — wrap the href via `formaters.ToFunctionalAssociationHref`
  exactly as the existing `links[]` builder does.

- **G2 — P4 [enhancement] `@link` carries only `href`.** Spec allows
  `rel`, `type`, `title`, `uid`, `hreflang`, `rt`, `if`. The `Link` struct
  in `internal/model/common_shared/links.go:35` already supports `rel`,
  `type`, `title`, `uid`. Populating `type: "application/json"` and
  `title` (the system's name) in `appendDatastreamAssociationLinks` would
  spare clients an extra fetch when rendering link previews. Pure UX win;
  zero spec risk.

- **G3 (already tracked in #1 §7.8)**: `Systems":null` capital-S field
  leaks into datastream JSON. Out-of-scope for #6.

## 7. Recommended action

Close issue #6 as **research-resolved with no implementation owed against
this issue**. Optionally file G1 and G2 as separate small enhancement
issues to capture the conformance polish work.

Comment posted on #6 — link to follow.

---

**Methodology note for this evaluation:** The headline lesson here is the
inverse of #4 ("plumbed-but-unconsumed model fields"). For #6, every model
field IS consumed and IS spec-conformant — the issue's premise was a client
expectation calibrated on a different (non-conformant) server, projected
onto the conformant one as a "missing feature". Future spikes that ask
"why doesn't server X look like server Y?" should start with the spec
inventory, not the comparative table. If the spec defines exactly one
shape, the answer to "should we add the other shape?" is almost always
"no, and the asymmetry is a defect on the side that lacks the conformant
shape."
