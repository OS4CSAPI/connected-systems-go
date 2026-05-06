# [PARKED] Report 13 — Strict JSON decoder rejects nested SensorML fields previously accepted; breaks OSHConnect-Python publisher bootstraps

> **⛔ PARKED 2026-05-05 — DO NOT FILE.**
>
> The framing of this report is **inverted**. Roundtrip testing on
> 2026-05-05 (POST `keywords` to pre-strict server, GET back) showed
> the pre-strict server returned 201 but **silently dropped** the
> field. Upstream `a467aba` ("Adding Strict Parsing") is therefore
> correct behavior surfacing pre-existing **silent SensorML field
> loss** in OSHConnect-Python publisher bootstraps — not an upstream
> regression.
>
> **Authoritative finding:** [`../issue-evaluations/silent-sensorml-field-loss-pre-strict-decoder.md`](../issue-evaluations/silent-sensorml-field-loss-pre-strict-decoder.md)
>
> **Disposition plan:** [`../plan-report-13-disposition.md`](../plan-report-13-disposition.md)
>
> **Bug location:** `OS4CSAPI/OSHConnect-Python` (publisher bootstraps
> emit `application/json` with GeoJSON-Feature shape carrying SensorML
> fields under `properties`). Fix shape: switch to
> `application/sml+json` with SensorML payload shape.
>
> The original report-13 body below is preserved for forensic value
> only. Do not act on its recommendations.

---

# Original report (parked — do not file)

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. There is **no companion `plan-NN`** for this report. The finding
>    surfaced during the 2026-05-05 deployment-pinning exercise (standing
>    up `cs-go-upstream` at upstream `df6da0d` and pointing the existing
>    OSHConnect-Python publisher fleet at it). It bypassed the plan-NN
>    advance-write stage because it was not visible from static
>    inspection alone — only from running an external client against
>    pinned upstream.
> 3. Re-read the live capture preserved inline in §1 / §3 of this report.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#17** (Strict decoder rejects nested SensorML fields; breaks OSHConnect-Python bootstraps) |
| Source fork issue | none — discovery finding, no fork-side issue |
| Research plan | none (see pre-work note) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P2** — breaking wire-protocol regression vs prior upstream; documented OSHConnect-Python publishers cannot bootstrap |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Defect persists on both static and live.

Strict-decoder commit pinned:

```text
$ git log --oneline c9747af..df6da0d -- \
    internal/api/procedure_handler.go \
    internal/model/formaters/geojson_formatters/procedure_geojson.go \
    internal/model/common_shared/

a467aba Adding Strict Parsing... now unknown fields will cause a bad
        request with the path and field name
c2ab201 time range and better 400 (now includes more info and field info)
1b2b614 Adding support for "latest" for TimeRange
fe9fbd0 Adding cascade delete and fixing existing cascade delete to full delete
635547f making the default limit for pagination configurable
```

Strict-decoder helper present on `df6da0d`:

```text
$ git grep -n 'func DecodeWithFieldErrors' df6da0d \
    -- internal/model/common_shared/

df6da0d:internal/model/common_shared/decode.go:33:
    func DecodeWithFieldErrors[T any](data []byte) (T, error) {
```

Live reproducer (cs-go-upstream, `df6da0d`):

```text
$ curl -s -i -X POST -u os4csapi:*** \
    -H 'Content-Type: application/json' \
    --data '{"type":"Feature","properties":{"uid":"urn:test:proc:1",
            "name":"Test","keywords":["a","b"]}}' \
    https://129-80-248-53.sslip.io/csapi-go-upstream/procedures

HTTP/2 400
content-type: application/json
content-length: 51

{"error":"unknown field 'keywords' in properties"}
```

Same payload against a deployment of upstream at `c9747af` (the parent
of the strict-decoder series — last upstream commit before `a467aba`):

```text
$ curl -s -i -X POST -u os4csapi:*** \
    -H 'Content-Type: application/json' \
    --data '{"type":"Feature","properties":{"uid":"urn:test:proc:1",
            "name":"Test","keywords":["a","b"]}}' \
    https://129-80-248-53.sslip.io/csapi-go/procedures

HTTP/2 201
location: https://.../csapi-go/procedures/353f50aa-9a4e-4e6a-8dc6-...
```

Same body. Same server path. **Upstream at `df6da0d` rejects with 400;
upstream at the parent of the strict-decoder series (`c9747af`) accepts
with 201.** The bisect window is the five commits listed above; the
inducing change is `a467aba` ("Adding Strict Parsing"). This is a
regression intra-upstream, not a fork-vs-upstream divergence.

**Conclusion:** As of `df6da0d`, the GeoJSON-Feature wrapper deserializer
for Procedure now uses `DecodeWithFieldErrors`, which rejects fields not
declared on the `Properties` struct of the wrapper. Procedure POSTs that
nest spec-legitimate fields (`keywords`, plus the others enumerated in §2)
under `properties` — the shape OSHConnect-Python and any pre-`a467aba`
client emits — get HTTP 400 deterministically.

> **Live-evidence caveat — both compared deployments are upstream
> builds.** `cs-go-upstream` is a fresh build of `upstream/main` at
> `df6da0d`. The second deployment (`cs-go`) is incidentally a build
> of the OS4CSAPI fork at fork commit `c9747af`, but the fork has not
> been modified — `c9747af` is byte-identical to a public upstream
> commit of the same SHA, reachable from `upstream/main` and serving
> here as a convenient "upstream at parent of `a467aba`" reproducer.
> The behavioural delta between the two endpoints is therefore
> attributable solely to the 98 upstream commits between `c9747af`
> and `df6da0d`, of which `a467aba` is the inducing one for this
> report. Reframed: the regression is **upstream vs prior upstream**,
> not fork vs upstream.

## 2. Static evidence

### 2.1 The strict-decode site

`internal/model/formaters/geojson_formatters/procedure_geojson.go`
(post-`a467aba`):

```go
func (f *ProcedureGeoJSONFormatter) Deserialize(
    ctx context.Context, reader io.Reader,
) (*domains.Procedure, error) {
    body, err := io.ReadAll(reader)
    if err != nil {
        return nil, err
    }
    geoJSON, err := common_shared.DecodeWithFieldErrors[struct {
        Type       string                             `json:"type"`
        ID         string                             `json:"id,omitempty"`
        Properties domains.ProcedureGeoJSONProperties `json:"properties"`
        Geometry   *common_shared.GoGeom              `json:"geometry,omitempty"`
        Links      common_shared.Links                `json:"links,omitempty"`
    }](body)
    if err != nil { … }
```

Parent commit (`c9747af`, fork HEAD) used a permissive decoder:

```go
var geoJSON struct { … }
if err := json.NewDecoder(reader).Decode(&geoJSON); err != nil { … }
```

`json.Decoder` without `DisallowUnknownFields()` ignores unknown JSON
keys silently. The new helper does not.

### 2.2 Where the rejected fields are actually declared

`internal/model/domains/procedure.go` — the **domain** struct (used for
GORM persistence and for the SensorML-JSON / SWE-JSON formatters):

```go
type Procedure struct {
    Base
    CommonSSN
    // …
    Keywords    common_shared.StringArray `json:"keywords,omitempty"`
    Identifiers common_shared.Terms       `json:"identifiers,omitempty"`
    Classifiers common_shared.Terms       `json:"classifiers,omitempty"`
    SecurityConstraints common_shared.SecurityConstraints
        `json:"securityConstraints,omitempty"`
    LegalConstraints    common_shared.LegalConstraints
        `json:"legalConstraints,omitempty"`
    Characteristics common_shared.CharacteristicGroups
        `json:"characteristics,omitempty"`
    Capabilities    common_shared.CapabilityGroups
        `json:"capabilities,omitempty"`
    Contacts        common_shared.ContactWrappers
        `json:"contacts,omitempty"`
    Documentation   common_shared.Documents     `json:"documentation,omitempty"`
    History         common_shared.History       `json:"history,omitempty"`
    // …
}
```

`internal/model/domains/procedure.go` — the **GeoJSON-wrapper Properties**
struct (`ProcedureGeoJSONProperties`, used **only** by the GeoJSON
formatter, around line 110+):

```go
type ProcedureGeoJSONProperties struct {
    UID         string   `json:"uid"`
    Name        string   `json:"name"`
    Description string   `json:"description,omitempty"`
    Keywords    []string `json:"keywords,omitempty"`   // declared
    // … fewer fields than the domain struct …
}
```

So `keywords` *is* declared on `ProcedureGeoJSONProperties` (line 114 in
`df6da0d`). The strict-decoder's "unknown field 'keywords' **in
properties**" error therefore cannot be a missing-field bug for
`keywords` itself — it is a **schema mismatch between the inner
generic struct literal in `procedure_geojson.go` and the named
`ProcedureGeoJSONProperties` struct**. The inner literal in §2.1 *uses*
`ProcedureGeoJSONProperties`, so this re-narrows to: the
`DecodeWithFieldErrors` walker is descending into nested struct types
and applying strict-mode there — and the named `Properties` struct's
field set does not match what historical clients send.

The 'keywords' rejection is the **first** field tripped by the OSHConnect
NWS bootstrap. The same client's payload also carries `identifiers`,
`classifiers`, `characteristics`, `capabilities`, `contacts`,
`documentation`, `history`, `securityConstraints`, `legalConstraints` —
all spec-legitimate SensorML fields declared on the **domain** struct
but **not** on `ProcedureGeoJSONProperties`. Each of those will trip the
same 400 once the prior field is removed. The report addresses the
**class** of regression, not the single-field case.

The same condition exists for Deployment and System — both have a
`{Type}GeoJSONProperties` companion struct that is a strict subset of
the domain struct (`git grep -n Keywords df6da0d -- internal/model/
domains/{deployment,system}.go` shows two declarations per file, the
domain one and the GeoJSON-properties one; the latter is a subset of
the former).

### 2.3 Inducing commit

`a467aba` ("Adding Strict Parsing... now unknown fields will cause a bad
request with the path and field name"). The series of five commits
listed in §1 are bisect-pinned via the file filter; `a467aba` is the
sole one that introduces strict-mode JSON decoding. The other four are
unrelated (`635547f` is pagination default, `fe9fbd0` is delete cascade,
`1b2b614`/`c2ab201` are TimeRange handling).

## 3. Live evidence

Source: live capture in §1 above. cs-go-upstream is a fresh-build pinned
deployment at upstream `df6da0d`, on Caddy at
`https://129-80-248-53.sslip.io/csapi-go-upstream/`, fronting Docker
container `csapi-upstream-api-1` (port 8284 → internal 8080).

```text
POST /procedures with {"properties":{"keywords":[…]}}    -> HTTP 400
                       body: {"error":"unknown field 'keywords' in properties"}
```

Cross-deployment confirmation (same body, deployment of upstream at
`c9747af` — the parent of the strict-decoder series; in this lab the
build happens to be the OS4CSAPI fork at fork commit `c9747af`, which
is byte-identical to the like-named upstream commit):

```text
POST /procedures with {"properties":{"keywords":[…]}}    -> HTTP 201
                       Location: /procedures/{uuid}
```

OSHConnect-Python `bootstrap_helpers.api_post` fails on this 400 (does
not retry — strict-mode 400s are non-recoverable), surfacing as:

```text
RuntimeError: HTTP 400 POST .../csapi-go-upstream/procedures:
{"error":"unknown field 'keywords' in properties"}
  at publishers/bootstrap_helpers.py:159 in fn
  at bootstrap_nws.py:511 in bootstrap (ensure_procedure call)
```

Source of the rejected field in client tooling — note this is a
spec-legitimate SensorML keyword list, not a vendor extension:

```text
$ grep -n keywords \
    publishers/nws/nws/bootstrap_nws.py
122:        "keywords": [
212:        "keywords": [
```

OSHConnect-Python `main` (used by all 10 publishers in the documented
fleet) emits `keywords` and the other SensorML metadata fields under
`properties` exactly as the SWE/SensorML JSON encoding has historically
been laid out — see Botts-Innovative-Research/OSHConnect-Python and the
OS4CSAPI fork at <https://github.com/OS4CSAPI/OSHConnect-Python>, both
shipping these payloads.

## 4. Spec authority

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001** — CSAPI Part 1: Feature Resources & Encodings | "OGC API - Connected Systems - Part 1: Feature Resources" under *OGC CSAPI Standards* | Defines the Procedure resource. The fields rejected by the strict decoder (`keywords`, `identifiers`, `classifiers`, `characteristics`, `capabilities`, `contacts`, `documentation`, `history`, `securityConstraints`, `legalConstraints`) are spec-legitimate Procedure metadata. The defect is not "client sent a non-spec field"; it is "server's GeoJSON-encoding wrapper omits properties the spec defines, then strict-mode rejects them." |
| **OGC 23-002** — CSAPI Part 2: Dynamic Data | "OGC API - Connected Systems - Part 2: Dynamic Data" under *OGC CSAPI Standards* | Same as 23-001 for Datastream/ControlStream/Observation/Command, where the same `*GeoJSONProperties`-vs-domain mismatch pattern likely repeats (out of scope for this report; flagged for follow-up audit). |
| **RFC 7493** — JSON Interchange Format §4.3 (Object Members) | "RFC 7493 — The I-JSON Message Format" under *IETF Specifications* | Permissive precedent: I-JSON `SHOULD` ignore unrecognized members. Strict-mode JSON decoding is policy, not compliance. |
| **OGC 19-072** — CSAPI Part 0: Core | "OGC API - Connected Systems - Part 0: Core" under *OGC CSAPI Standards* | Backwards-compatibility expectations for documented client tooling that targets the spec. |

**Out-of-scope sources (not cited in the public extract):**

- OAS 3.0.3 — irrelevant; no schema-document component is being violated.
- W3C SensorML XML — informative only; the JSON encoding is what is in
  scope here.

## 5. Alternatives considered (internal-only)

Three viable fix shapes:

### Option A — Make `*GeoJSONProperties` complete (recommended)

Synchronise `ProcedureGeoJSONProperties`,
`DeploymentGeoJSONProperties`, and `SystemGeoJSONProperties` with the
field set of their corresponding domain struct, so every spec-legitimate
field declared on the domain struct is also declared on the
GeoJSON-properties struct. The strict decoder then accepts every
spec-legitimate field, and historical clients that include them
continue to work.

- **Authoring delta:** ~30 declarative struct-field additions across
  three files. No logic changes.
- **Risk:** Low. Each added field's type is the same as on the domain
  struct (already the source of truth for serialization).
- **Spec fidelity:** Restores it. The current state is **less**
  spec-faithful, not more.

### Option B — Loosen the GeoJSON-wrapper decode to permissive

Special-case the GeoJSON wrapper deserializers to fall back to
`json.NewDecoder(reader).Decode(&geoJSON)` (no
`DisallowUnknownFields()`).

- **Authoring delta:** trivial (revert the four formatter sites that
  switched to `DecodeWithFieldErrors` to the prior permissive decode).
- **Risk:** Medium — it surrenders the strict-mode guard for an entire
  encoding family. `a467aba`'s headline benefit was exactly that
  guard; reverting it for one encoding undermines the rationale of
  that commit.
- **Spec fidelity:** Neutral. No worse than pre-`a467aba`.

### Option C — Accept the regression; require clients to update

Document the breaking change; require all clients to drop fields the
GeoJSON-properties struct does not currently declare.

- **Authoring delta:** zero on server side.
- **Risk:** High for ecosystem health. OSHConnect-Python is the
  maintained Python client surfaced in OGC Connected Systems demos.
  Forcing it (and other downstream tooling) to drop spec-legitimate
  fields converts a server-side oversight into ecosystem churn.

**Ranking: Option A.** Lead with A only; B is a strategic retreat from
`a467aba`'s headline guard, and C off-loads upstream cost onto
documented downstream clients for what is materially a server-side
encoding asymmetry.

## 6. Recommended fix

**Lead with Option A.** Sync `ProcedureGeoJSONProperties`,
`DeploymentGeoJSONProperties`, and `SystemGeoJSONProperties` with their
corresponding domain structs. The minimum viable change for Procedure:

```go
type ProcedureGeoJSONProperties struct {
    UID         string   `json:"uid"`
    Name        string   `json:"name"`
    Description string   `json:"description,omitempty"`
    // — additions to match domain struct —
    Keywords            common_shared.StringArray            `json:"keywords,omitempty"`
    Identifiers         common_shared.Terms                  `json:"identifiers,omitempty"`
    Classifiers         common_shared.Terms                  `json:"classifiers,omitempty"`
    Characteristics     common_shared.CharacteristicGroups   `json:"characteristics,omitempty"`
    Capabilities        common_shared.CapabilityGroups       `json:"capabilities,omitempty"`
    Contacts            common_shared.ContactWrappers        `json:"contacts,omitempty"`
    Documentation       common_shared.Documents              `json:"documentation,omitempty"`
    History             common_shared.History                `json:"history,omitempty"`
    SecurityConstraints common_shared.SecurityConstraints    `json:"securityConstraints,omitempty"`
    LegalConstraints    common_shared.LegalConstraints       `json:"legalConstraints,omitempty"`
    // (similar additions for deployment and system as their domain
    //  structs dictate — audit each file)
}
```

Equivalent additive sync for Deployment and System.

Implementation surface: three files —
`internal/model/domains/{procedure,deployment,system}.go` — each one
struct.

## 7. Scope guard

What NOT to touch as part of this filing:

- The `DecodeWithFieldErrors` helper itself. The strict-mode behaviour
  is the desirable headline of `a467aba` and should remain.
- The domain structs (already correct).
- The four Procedure handler sites that route to the formatter
  (the writeDeserializeError refactor in `a467aba` is fine).
- The other resource types (Datastream / ControlStream / Observation /
  Command / SamplingFeature) — they have their own
  `*GeoJSONProperties` structs with their own audit. Out of scope here;
  flag as a follow-up audit (backlog item separate).
- Any other formatter family (SensorML-JSON, SWE-JSON). Those use the
  full domain struct directly; they are not affected.

## 8. Fork-side context (internal-only)

- Found during the 2026-05-05 deployment-pinning exercise: standing up
  `cs-go-upstream` at `df6da0d` next to the existing fork-build
  `cs-go` (`c9747af`) and `cs-go-head` (`4b994212`), then attempting
  to fan the OSHConnect-Python publisher fleet at the new endpoint.
- The user's standing constraint that the fork has not been modified
  was verified during this finding — `c9747af` is reachable from
  `upstream/main` (the fork at that SHA is a clean snapshot, no
  fork-side patches), see §1 for the bisect listing. The fork's source
  tree on `origin/main` *does* contain `a467aba` (via merge commit
  `22e8bcd` "Merge upstream SomethingCreativeStudios/main into
  OS4CSAPI fork"); the in-the-loop `cs-go` deployment is simply a
  stale binary built before that merge, which is why it serves as a
  pre-`a467aba` reproducer.
- The behavioural delta documented in §1 is therefore
  upstream-vs-prior-upstream (pre- and post-`a467aba`), not
  fork-vs-upstream.
- The publisher fleet currently runs against the stale `cs-go` and
  `cs-go-head` deployments (both at pre-`a467aba` SHAs) without
  issue. They cannot bootstrap against the upstream-pinned
  `cs-go-upstream` deployment at `df6da0d`, hence this report. We
  will not patch the publishers around the upstream regression — they
  are documented OSHConnect tooling and should remain spec-aligned.
- The placeholder seed (3 systems / 3 datastreams / 3 observations /
  1 controlstream / 1 command / 1 deployment) created via direct
  field-stripped POST in `seed-upstream.sh` is a workaround for our
  own report-stream evidence-gathering needs; it is **not** evidence
  the regression is fixable client-side at large.

## 9. Open questions resolved

| Question | Resolution |
|---|---|
| Is `keywords` actually unknown to the model, or just unknown to the wrapper? | **Just the wrapper.** It is declared on `ProcedureGeoJSONProperties` (`procedure.go` ~line 114 on `df6da0d`); the strict decoder's path-prefixed error narrows it to a different nested struct in `procedure_geojson.go`. See §2.2. The framing — "wrapper is a strict subset of the domain struct" — generalises to the other rejected fields. |
| Severity P2 or P3? | **P2.** Documented client tooling (OSHConnect-Python publisher fleet) cannot bootstrap; OGC ecosystem demo posture broken. |
| Should we also flag the `properties` "uid" rejection on Datastream et al. (report-02 behaviour)? | **No, separate filing.** Report-02 already covers the Datastream side of the strict-mode question for `uid` specifically. This report is the **broader** wrapper-vs-domain class. The interaction between the two will make sense to the maintainer because both reports cite `a467aba`. |
| Lead with Option A or B? | **A only.** B undermines `a467aba`'s headline; A restores spec fidelity. |
| Cite the OSHConnect-Python publisher repo? | **Yes** — both Botts-Innovative-Research/OSHConnect-Python and the OS4CSAPI fork. The maintainer needs to see the rejection has a real, documented client cost. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P2] Strict JSON decoder rejects spec-legitimate SensorML metadata fields nested under properties (Procedure, Deployment, System); breaks documented OSHConnect-Python publishers`

**Labels:** `bug`, `regression`

---

### Context

`a467aba` ("Adding Strict Parsing... now unknown fields will cause a
bad request with the path and field name") introduced
`common_shared.DecodeWithFieldErrors[T]` and switched the GeoJSON-wrapper
deserializers (Procedure / Deployment / System) to use it. The intent
of the commit is good: surface payload mistakes early with a
field-named 400. Adopting this is a correct step.

However: each `{Resource}GeoJSONProperties` struct is a **strict
subset** of its corresponding domain struct in
`internal/model/domains/`. The domain struct declares the full
SensorML/SWE-JSON metadata field set (`keywords`, `identifiers`,
`classifiers`, `characteristics`, `capabilities`, `contacts`,
`documentation`, `history`, `securityConstraints`, `legalConstraints`,
…). The GeoJSON-properties wrapper declares roughly only the headline
trio (`uid`, `name`, `description`) plus a couple of scalars.

After `a467aba`, any POST to `/procedures` (and same for `/deployments`,
`/systems`) whose `properties` object includes a SensorML-legitimate
metadata field that was historically accepted now returns:

```text
HTTP 400
{"error":"unknown field 'keywords' in properties"}
```

This is a **breaking wire-protocol regression** vs the immediate
parent of `a467aba`. Documented OSHConnect-Python publishers
(Botts-Innovative-Research/OSHConnect-Python and the OS4CSAPI fork at
<https://github.com/OS4CSAPI/OSHConnect-Python> — the maintained Python
client used in OGC Connected Systems demos and a 10-publisher
real-time fleet) cannot bootstrap their procedures, deployments, or
systems against a fresh-built `upstream/main` deployment as a result.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

Three GeoJSON-properties wrapper structs
(`ProcedureGeoJSONProperties`, `DeploymentGeoJSONProperties`,
`SystemGeoJSONProperties`) are missing field declarations that exist
on their corresponding domain structs. After `a467aba`'s adoption of
`DecodeWithFieldErrors`, this gap becomes a hard 400 on POST/PUT for
spec-legitimate fields that the same server **would have accepted at
the parent commit** and **does still serialize back out** through the
SensorML-JSON / SWE-JSON formatters.

This is server-side: clients are sending fields the spec sanctions and
the model already supports.

### Static evidence

`internal/model/formaters/geojson_formatters/procedure_geojson.go`
(post-`a467aba`):

```go
geoJSON, err := common_shared.DecodeWithFieldErrors[struct {
    Type       string                             `json:"type"`
    ID         string                             `json:"id,omitempty"`
    Properties domains.ProcedureGeoJSONProperties `json:"properties"`
    Geometry   *common_shared.GoGeom              `json:"geometry,omitempty"`
    Links      common_shared.Links                `json:"links,omitempty"`
}](body)
```

`internal/model/domains/procedure.go` — domain struct (full SensorML
field set, used by SensorML-JSON / SWE-JSON formatters and GORM):

```go
type Procedure struct {
    Base; CommonSSN
    // …
    Keywords            common_shared.StringArray         `json:"keywords,omitempty"`
    Identifiers         common_shared.Terms               `json:"identifiers,omitempty"`
    Classifiers         common_shared.Terms               `json:"classifiers,omitempty"`
    Characteristics     common_shared.CharacteristicGroups `json:"characteristics,omitempty"`
    Capabilities        common_shared.CapabilityGroups     `json:"capabilities,omitempty"`
    Contacts            common_shared.ContactWrappers      `json:"contacts,omitempty"`
    Documentation       common_shared.Documents            `json:"documentation,omitempty"`
    History             common_shared.History              `json:"history,omitempty"`
    SecurityConstraints common_shared.SecurityConstraints  `json:"securityConstraints,omitempty"`
    LegalConstraints    common_shared.LegalConstraints     `json:"legalConstraints,omitempty"`
    // …
}
```

`internal/model/domains/procedure.go` — `ProcedureGeoJSONProperties`
(strict subset, used **only** by the GeoJSON-wrapper deserializer):

```go
type ProcedureGeoJSONProperties struct {
    UID         string   `json:"uid"`
    Name        string   `json:"name"`
    Description string   `json:"description,omitempty"`
    Keywords    []string `json:"keywords,omitempty"`
    // (most other SensorML metadata fields declared on the domain
    //  struct are not present here)
}
```

The strict decoder's path-prefixed error message (`"unknown field
'keywords' in properties"`) narrows the rejection to the nested
`Properties` struct above — which, despite declaring a `Keywords`
field, evidently does not match the historical client payload shape
through the strict walker. Stripping `keywords` exposes the next
unknown field (`identifiers`, etc.) — repeat for every domain-only
field, and the eventual minimum-viable payload is
`{uid, name, description}` only. That payload set is materially
narrower than the SensorML metadata clients have historically sent.

Bisect:

```text
$ git log --oneline c9747af..df6da0d -- \
    internal/api/procedure_handler.go \
    internal/model/formaters/geojson_formatters/procedure_geojson.go \
    internal/model/common_shared/

a467aba Adding Strict Parsing... now unknown fields will cause a bad
        request with the path and field name             <-- inducing
c2ab201 time range and better 400 (now includes more info and field info)
1b2b614 Adding support for "latest" for TimeRange
fe9fbd0 Adding cascade delete and fixing existing cascade delete to full delete
635547f making the default limit for pagination configurable
```

The other four commits in the window are unrelated. `a467aba` is the
sole inducing change.

### Live evidence

Two upstream-build endpoints, identical request body:

```text
$ curl -s -i -X POST -u user:*** \
    -H 'Content-Type: application/json' \
    --data '{"type":"Feature","properties":{"uid":"urn:test:proc:1",
            "name":"Test","keywords":["a","b"]}}' \
    https://.../csapi-go-upstream/procedures      # upstream df6da0d
                                                  # (post-a467aba)

HTTP/2 400
{"error":"unknown field 'keywords' in properties"}

$ curl -s -i -X POST -u user:*** \
    -H 'Content-Type: application/json' \
    --data '{"type":"Feature","properties":{"uid":"urn:test:proc:1",
            "name":"Test","keywords":["a","b"]}}' \
    https://.../csapi-go/procedures               # upstream at c9747af
                                                  # (parent of a467aba)

HTTP/2 201
location: /procedures/353f50aa-9a4e-4e6a-8dc6-…
```

Both endpoints serve upstream code; the second is at the immediate
parent of `a467aba`. The regression is intra-upstream.

OSHConnect-Python NWS publisher bootstrap, against upstream `df6da0d`:

```text
RuntimeError: HTTP 400 POST .../procedures:
{"error":"unknown field 'keywords' in properties"}
  at publishers/bootstrap_helpers.py:159 in fn
  at bootstrap_nws.py:511 in bootstrap (ensure_procedure call)
```

### Recommended fix

Sync each `*GeoJSONProperties` wrapper struct with the field set of its
corresponding domain struct. Three files — one struct each:

- `internal/model/domains/procedure.go` —
  `ProcedureGeoJSONProperties`
- `internal/model/domains/deployment.go` —
  `DeploymentGeoJSONProperties`
- `internal/model/domains/system.go` —
  `SystemGeoJSONProperties`

For each missing domain field that is part of the SensorML / SWE-JSON
spec for that resource, add a matching declaration with the same JSON
tag and an aligned type. No logic changes; declarative additions only.

This restores acceptance of spec-legitimate fields under
`properties` while preserving `a467aba`'s strict-mode guard against
genuine typos.

### Spec authority

- **OGC 23-001** (CSAPI Part 1 — Feature Resources & Encodings): defines
  the Procedure / System / Deployment resource shape, including the
  SensorML metadata fields rejected here.
- **OGC 23-002** (CSAPI Part 2 — Dynamic Data): same wrapper-vs-domain
  pattern likely repeats for the dynamic-data resource types
  (Datastream / ControlStream / Observation / Command); flagged as a
  separate audit.
- **RFC 7493 §4.3** (I-JSON): "implementations SHOULD ignore unknown
  members" — informative; supports A's framing as the spec-faithful
  fix.

### Severity

**P2** — breaking wire-protocol regression vs the parent commit;
documented client tooling (OSHConnect-Python) cannot bootstrap;
maintained OS4CSAPI publisher fleet (10 services covering NWS / NDBC /
CO-OPS / ISS / OpenSky / USGS / Aviation Weather data sources) is
blocked.

---

**Validation chain:** [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) #17 → this report → public-facing extract above.
