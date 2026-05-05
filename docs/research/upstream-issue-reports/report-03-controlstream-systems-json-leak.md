# Report 03 — ControlStream `Systems` field JSON leak

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-03-controlstream-systems-json-leak.md`](../upstream-issues/plan-03-controlstream-systems-json-leak.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#5** (ControlStream `Systems` field JSON leak) |
| Source fork issue | `OS4CSAPI/connected-systems-go#15` (closed; sibling defect deferred to separate filing per #15's own scope rule) |
| Research plan | [`../upstream-issues/plan-03-controlstream-systems-json-leak.md`](../upstream-issues/plan-03-controlstream-systems-json-leak.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P3-Minor** — matches parent issue #15 triage; corrects backlog's "P2 — sibling" entry (cannot exceed parent severity) |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Defect persists on both static and live.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git show upstream/main:internal/model/domains/control_stream.go |
    Select-String -Pattern 'Systems\s+\[\]System' -Context 2,2

      SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`

>     Systems []System `gorm:"many2many:system_controlstreams;"`
  }
```

```text
$ git show upstream/main:internal/model/domains/datastream.go |
    Select-String -Pattern 'Systems\s+\[\]System' -Context 1,1

>     Systems []System `gorm:"many2many:system_datastreams;" json:"-"`
  }
```

```text
$ git show upstream/main:internal/model/domains/control_stream.go |
    Select-String -Pattern 'MarshalJSON'
(no matches — confirms json:"-" is sufficient)

$ git show upstream/main:internal/model/domains/control_stream.go |
    Select-String -Pattern 'json:"-"'
SystemID            *string `gorm:"type:varchar(255);index" json:"-"`
ProcedureID         *string `gorm:"type:varchar(255);index" json:"-"`
DeploymentID        *string `gorm:"type:varchar(255);index" json:"-"`
FeatureOfInterestID *string `gorm:"type:varchar(255);index" json:"-"`
SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`
(every sibling FK field carries json:"-" — internal convention)

$ git log --oneline upstream/main -- internal/model/domains/control_stream.go |
    Select -First 5
d2d1347 Cleaned up data/controlstream's SystemLink
dacae7b Bug fixes and clean up
f2cf1c3 Adding the other resources along with e2e tests
(no commit since eval has touched the Systems field)
```

```text
$ curl.exe -s '…/csapi-go-upstream/controlstreams/0e349016-…'

{"id":"0e349016-be02-4d85-961d-d94b7fc3323e","uid":"","name":"Thermostat CS A",
 "formats":["application/json"],"inputName":"thermostat-set-point",
 "schema":{…},
 "links":[{"href":"…/controlstreams/0e349016-…/commands","rel":"ogc-rel:commands"}],
 "Systems":null,                                             <-- the leak
 "system@link":{"href":"…/systems/ced04f76-…"}}

$ curl.exe -s '…/csapi-go-upstream/datastreams/35a750f9-…'

{"id":"35a750f9-…","name":"Temperature DS A","description":"Seeded datastream",
 "formats":["application/json"],"outputName":"temperature","type":"observation",
 "schema":{…},
 "links":[…],
 "system@link":{"href":"…/systems/ced04f76-…"}}                <-- no "Systems" key
```

**Conclusion:** As of `df6da0d`, `ControlStream.Systems` still lacks
`json:"-"` and serialises as a top-level `"Systems": null` member in
every controlstream HTTP response. The parent fix landed on Datastream
in `2dc09f7` exactly as intended; the same one-line change has not
been applied to the sibling struct. Defect is live on a fresh
upstream-pinned deployment.

> **Live-evidence note.** The two captures above are taken **from the
> same database, same Caddy host, same upstream-build deployment
> (`csapi-go-upstream` at `df6da0d`)**, on resources seeded minutes
> earlier from a single bootstrap script. The structural difference
> between the two response bodies is therefore attributable solely to
> the divergent struct-tag convention between `Datastream.Systems`
> (carries `json:"-"`) and `ControlStream.Systems` (does not).

## 2. Static evidence

Source: [`../evidence/issue-015/static-analysis-2026-04-30.md`](../evidence/issue-015/static-analysis-2026-04-30.md) §3.4 (refreshed in §1 above).

`internal/model/domains/control_stream.go`, `ControlStream` struct
(post-`d2d1347`):

```go
type ControlStream struct {
    Base

    // … (other fields)

    SystemID            *string `gorm:"type:varchar(255);index" json:"-"`
    ProcedureID         *string `gorm:"type:varchar(255);index" json:"-"`
    DeploymentID        *string `gorm:"type:varchar(255);index" json:"-"`
    FeatureOfInterestID *string `gorm:"type:varchar(255);index" json:"-"`
    SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`

    Systems []System `gorm:"many2many:system_controlstreams;"`     // <-- no json:"-"
}
```

`internal/model/domains/datastream.go`, `Datastream` struct
(post-`2dc09f7`, the parent fix):

```go
type Datastream struct {
    Base

    // … (other fields)

    Systems []System `gorm:"many2many:system_datastreams;" json:"-"`   // <-- has it
}
```

Convention sweep on `ControlStream`: every other foreign-key field on
the struct carries `json:"-"` (five sibling FK fields enumerated above).
The `Systems` many2many field is the **sole** outlier on
`ControlStream`. There is no `MarshalJSON` method on `ControlStream`,
so `json:"-"` is the correct and sufficient mechanism — exactly as on
the parent.

`d2d1347` ("Cleaned up data/controlstream's SystemLink") touched both
files in tandem when reorganising `system@link` projection. `2dc09f7`
("oh gorm/go… adding `json:-`") was the explicit fix on the Datastream
side. The ControlStream side was missed during `2dc09f7`.

## 3. Live evidence

Source: live capture in §1 above. cs-go-upstream is a fresh-build
upstream-pinned deployment at `df6da0d`, fronted by Caddy at
`https://129-80-248-53.sslip.io/csapi-go-upstream/`.

```text
GET /controlstreams/0e349016-be02-4d85-961d-d94b7fc3323e   -> 200, body contains
                                                              "Systems": null
GET /controlstreams?limit=2                                -> 200, every item
                                                              contains
                                                              "Systems": null
GET /datastreams/35a750f9-…                                -> 200, body contains
                                                              no "Systems" key
```

The ControlStream and Datastream resources sit in the same database and
are returned by structurally similar handler/formatter pipelines. The
behavioural difference between the two response bodies is the
single-line struct tag.

## 4. Spec authority

This finding has the same spec posture as parent issue #15: it is
**not a strict schema violation**, but it is an RFC 7493 SHOULD-grade
interoperability defect and an internal-convention defect.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 `controlStream.json` / `baseStream.json` (Part 2 OpenAPI bundle) | "OGC API - Connected Systems - Part 2: Dynamic Data" and "OGC API - Connected Systems - Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Establishes canonical ControlStream property set. None named `Systems` (any case). Schemas do not declare `additionalProperties: false`, so the leak is permissive-not-prohibited. |
| **RFC 7493** §4.3 (I-JSON) | "RFC 7493 — The I-JSON Message Format" under *IETF Specifications* | The interoperability SHOULD: senders that wish to be widely interoperable SHOULD only emit members defined in the protocol. Same SHOULD invoked for parent issue #15. |
| **JSON Schema 2020-12** `additionalProperties` default semantics | "JSON Schema Validation 2020-12" under *JSON / OAS* | Backstop for "permissive-not-prohibited" framing; explains why this is a SHOULD not a SHALL. |

**Out-of-scope sources (not cited in the public extract):**

- OAS 3.0.3 — not the binding standard for response-body content rules here.
- OGC 23-001 (Part 1) — defines neither Datastream nor ControlStream.
- OGC 19-072 (OGC API – Common) — not relevant; serialization concern,
  not landing-page / API-definition concern.

## 5. Alternatives considered (internal-only)

Only one viable fix shape — the one already established as upstream
precedent.

### Option A — Append `json:"-"` to the existing struct tag (recommended)

```go
Systems []System `gorm:"many2many:system_controlstreams;" json:"-"`
```

- **Authoring delta:** one-line change (append ` json:"-"` to one tag).
- **Risk:** zero. Identical change to `2dc09f7` on `Datastream.Systems`,
  already merged and behaviourally validated.
- **Spec fidelity:** restores it. The current state emits a property
  not declared in CSAPI Part 2.

### Options B and onward

None considered. The maintainer has already accepted Option A's exact
shape on the parent; no architectural alternative is on the table.

**Ranking: Option A.** This is a precedent-bound one-liner.

## 6. Recommended fix

**Lead with Option A** — append `json:"-"` to the `Systems` field:

```go
type ControlStream struct {
    // …
    Systems []System `gorm:"many2many:system_controlstreams;" json:"-"`
}
```

Implementation surface:
`internal/model/domains/control_stream.go` — single struct-tag append
on the `Systems` field. Mirrors `2dc09f7` on `Datastream.Systems`
exactly.

## 7. Scope guard

What NOT to touch as part of this filing:

- The `Systems` field name itself. Lower-cased `systems` is **not** in
  the canonical Part 2 schema either; renaming would not improve spec
  fidelity. The fix is to omit, not to rename.
- The other `ControlStream` fields. All five sibling FK fields already
  carry `json:"-"`. No convention sweep is needed beyond the headline
  one-liner.
- The six latent untagged relationship slices on `System` and
  `Procedure` enumerated in eval §3.5. Those are tracked separately
  as backlog item #6 (optional hardening; conditional on maintainer
  appetite). Out of scope here.
- GORM tag changes. The `many2many:system_controlstreams` join is
  correct and required. Only the JSON tag is missing.
- Any `MarshalJSON` introduction. Unnecessary; `json:"-"` is
  sufficient and matches parent precedent.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#15` was filed against the parent
  defect on `Datastream.Systems`. Maintainer accepted, triaged
  P3-Minor, and shipped `2dc09f7`. Issue #15's own scope rule —
  "If you find more, file separately, one issue per leaking type" —
  explicitly defers the ControlStream sibling to this filing.
- Backlog labels this **P2 — sibling of fix that landed on
  Datastream**. The eval at
  [`../issue-evaluations/issue-015.md`](../issue-evaluations/issue-015.md)
  §3.6 explicitly concludes "P3-Minor is exactly right. P2 would imply
  functional or strict-spec impact; neither exists." Sibling cannot
  exceed parent severity. Backlog entry will be corrected to
  P3-Minor in tandem with this filing.
- The OSHConnect-Python publisher fleet (cf. report-13) does not
  POST to `/controlstreams` in any current bootstrap, so the leak
  has not surfaced operationally on the fork's live deployments.
  Risk profile remains the same regardless: clients that consume
  controlstream responses see a non-canonical field they must
  defensively ignore.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Severity declaration — backlog says P2, eval and parent say P3-Minor. | **P3-Minor.** Sibling cannot exceed parent triage. Backlog flag from plan §1 confirmed; backlog entry to be updated in tandem with this filing. |
| Cite parent commit `2dc09f7` as precedent? | **Yes — lead with it.** Justifies both spec posture and fix shape from the maintainer's own merged work. |
| Bundle §3.5 latent-slices hardening? | **No.** Out-of-scope mention only; backlog item #6 is the dedicated tracker. |
| Live evidence required, or is static-only sufficient? | **Live captured this round.** cs-go-upstream had a controlstream available from the placeholder seed; live capture above shows the leak deterministically. |
| Cite eval/evidence paths? | **Yes**, as "validation chain" footer links. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P3-Minor] ControlStream JSON response leaks "Systems": null — sibling defect of #15 / 2dc09f7 fix`

**Labels:** `bug`

---

### Context

`2dc09f7` ("oh gorm/go… adding `json:-`") fixed the parent defect on
`Datastream.Systems` per issue #15 — appending `json:"-"` to the
GORM many2many field so it stops serialising into HTTP response
bodies. Issue #15's own scope rule was "one issue per leaking type;
file siblings separately." This is the dedicated sibling filing for
`ControlStream.Systems`.

The same untagged field exists on `ControlStream` and was not
updated in the same commit. Every controlstream HTTP response
therefore emits a top-level `"Systems": null` member that is not
defined in CSAPI Part 2's `controlStream.json` / `baseStream.json`
schemas.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

`internal/model/domains/control_stream.go` `ControlStream` carries:

```go
Systems []System `gorm:"many2many:system_controlstreams;"`
```

with no `json:"-"` tag, causing `encoding/json` to emit `"Systems": null`
(capital-S, top-level) in every `GET /controlstreams` and
`GET /controlstreams/{id}` response. The parent struct
`Datastream.Systems` was given exactly that tag in `2dc09f7`; the
sibling was missed.

### Static evidence

`internal/model/domains/control_stream.go` (post-`d2d1347`):

```go
type ControlStream struct {
    Base
    // …
    SystemID            *string `gorm:"type:varchar(255);index" json:"-"`
    ProcedureID         *string `gorm:"type:varchar(255);index" json:"-"`
    DeploymentID        *string `gorm:"type:varchar(255);index" json:"-"`
    FeatureOfInterestID *string `gorm:"type:varchar(255);index" json:"-"`
    SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`

    Systems []System `gorm:"many2many:system_controlstreams;"`   // missing json:"-"
}
```

For comparison, `internal/model/domains/datastream.go`
(post-`2dc09f7`):

```go
Systems []System `gorm:"many2many:system_datastreams;" json:"-"`
```

Every other foreign-key field on `ControlStream` carries `json:"-"`.
The `Systems` many2many field is the sole outlier. There is no
`MarshalJSON` method on `ControlStream`, so `json:"-"` is the
correct and sufficient mechanism — same as on the parent.

### Live evidence

Same database, same upstream-pinned build, same Caddy host:

```text
$ curl -s '…/controlstreams/0e349016-be02-4d85-961d-d94b7fc3323e'
{
  "id": "0e349016-…",
  "uid": "",
  "name": "Thermostat CS A",
  "formats": ["application/json"],
  "inputName": "thermostat-set-point",
  "schema": {…},
  "links": [{"href":"…/controlstreams/0e349016-…/commands","rel":"ogc-rel:commands"}],
  "Systems": null,                                            <-- the leak
  "system@link": {"href":"…/systems/ced04f76-…"}
}

$ curl -s '…/datastreams/35a750f9-…'
{
  "id": "35a750f9-…",
  "name": "Temperature DS A",
  …
  "system@link": {"href":"…/systems/ced04f76-…"}              <-- no "Systems" key
}
```

The structural difference between the two response bodies on the same
deployment is the single-line struct-tag divergence.

### Recommended fix

Append `json:"-"` to the `Systems` field, mirroring `2dc09f7` exactly:

```go
Systems []System `gorm:"many2many:system_controlstreams;" json:"-"`
```

Implementation surface: one struct-tag append in
`internal/model/domains/control_stream.go`. No logic changes, no
migration changes.

### Spec authority

- **OGC 23-002** — CSAPI Part 2 `controlStream.json` /
  `baseStream.json` define the canonical ControlStream property set.
  No member named `Systems` (any case). Schemas do not declare
  `additionalProperties: false`, so the leak is permissive-not-prohibited.
- **RFC 7493 §4.3** (I-JSON) — interoperability SHOULD: senders SHOULD
  only emit members defined in the protocol. Same SHOULD invoked for
  parent issue #15.

### Severity

**P3-Minor** — matches parent issue #15 triage. No functional impact;
clients that consume controlstream responses must defensively ignore
the non-canonical key.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-015.md`](../issue-evaluations/issue-015.md) §3.4, §3.6, §3.7
- Evidence: [`docs/research/evidence/issue-015/static-analysis-2026-04-30.md`](../evidence/issue-015/static-analysis-2026-04-30.md) §3.4
- Plan: [`docs/research/upstream-issues/plan-03-controlstream-systems-json-leak.md`](../upstream-issues/plan-03-controlstream-systems-json-leak.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #5
