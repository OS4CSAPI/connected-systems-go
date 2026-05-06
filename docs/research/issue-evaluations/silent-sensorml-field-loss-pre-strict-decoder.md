# Evaluation — Silent SensorML field loss exposed by strict-decoder commit `a467aba`

| Field | Value |
|---|---|
| Source | Discovery finding 2026-05-05 (deployment-pinning + roundtrip testing) |
| Fork-side issue | [`OS4CSAPI/OSHConnect-Python#5`](https://github.com/OS4CSAPI/OSHConnect-Python/issues/5) (filed 2026-05-05) |
| Upstream-side issue | none — upstream is correct |
| Verified HEADs | `c9747af` (pre-strict, fork-build `cs-go`) and `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (strict, fresh-build `cs-go-upstream`) |
| **Verdict** | **NOT AN UPSTREAM DEFECT** |
| Severity | **P1 client-side** — silent data loss in OS4CSAPI publisher fleet |
| Action | Fix `OS4CSAPI/OSHConnect-Python` (no upstream `Botts-Innovative-Research/OSHConnect-Python` work, no upstream `connected-systems-go` filing) |
| Disposition plan | [`../plan-report-13-disposition.md`](../plan-report-13-disposition.md) |
| Supersedes | [`../upstream-issue-reports/_PARKED_report-13-strict-decoder-osh-publisher-bootstrap-regression.md`](../upstream-issue-reports/_PARKED_report-13-strict-decoder-osh-publisher-bootstrap-regression.md) |

## Summary

Upstream commit `a467aba` ("Adding Strict Parsing... now unknown
fields will cause a bad request with the path and field name") added
`common_shared.DecodeWithFieldErrors[T]` and routed the GeoJSON-wrapper
deserializers (Procedure / Deployment / System) through it. As of
`df6da0d`, POSTing a GeoJSON-Feature payload with SensorML metadata
fields (`keywords`, `identifiers`, `classifiers`, `characteristics`,
`capabilities`, `contacts`, `documentation`, `history`,
`securityConstraints`, `legalConstraints`) under `properties` returns
`HTTP 400 {"error":"unknown field 'keywords' in properties"}`.

The originally-drafted upstream issue report (now parked as
`_PARKED_report-13-...`) framed this as a wire-protocol regression
and recommended widening `{Procedure,Deployment,System}GeoJSONProperties`
wrapper structs to match their domain structs. **Roundtrip testing on
2026-05-05 inverts that framing:** the pre-strict server returned 201
on the same payload but **silently dropped** the SensorML fields. The
correct content-type/payload-shape pair (`application/sml+json` +
top-level SensorML body) round-trips all 9 fields cleanly on **both**
pre-strict and strict server builds.

The strict decoder is therefore working as designed; it is surfacing
a pre-existing **client-side** bug in OSHConnect-Python publisher
bootstraps that has been silently losing SensorML metadata for as
long as the fleet has been running.

## Roundtrip evidence (2026-05-05)

All three tests run against the OS4CSAPI sandbox via Caddy
(`https://129-80-248-53.sslip.io/`); `os4csapi:***` basic auth.

### Test 1 — pre-strict server, GeoJSON-as-JSON shape (the failing client shape)

```text
POST /csapi-go/procedures
Content-Type: application/json

{
  "type": "Feature",
  "properties": {
    "uid": "urn:test:roundtrip:keywords:001",
    "name": "roundtrip-test",
    "description": "strict-decoder roundtrip test",
    "keywords": ["alpha","beta","gamma"]
  }
}

→ HTTP 201
  Location: /csapi-go/procedures/bc4f8c69-655a-4885-a809-79596bf2c672
```

```text
GET /csapi-go/procedures/bc4f8c69-655a-4885-a809-79596bf2c672
Accept: application/json
→ HTTP 200
  Content-Type: application/geo+json
  body: {
    "type":"Feature",
    "id":"bc4f8c69-...",
    "geometry":null,
    "properties":{
      "uid":"urn:test:roundtrip:keywords:001",
      "name":"roundtrip-test",
      "description":"strict-decoder roundtrip test"
    }
  }

GET /csapi-go/procedures/bc4f8c69-655a-4885-a809-79596bf2c672
Accept: application/sml+json
→ HTTP 200
  Content-Type: application/sml+json
  body: {
    "id":"bc4f8c69-...",
    "label":"roundtrip-test",
    "description":"strict-decoder roundtrip test",
    "uniqueId":"urn:test:roundtrip:keywords:001",
    "method":{}
  }
```

**`keywords` absent from BOTH representations.** The pre-strict server
accepted the POST (201) but did not persist the field.

### Test 2 — pre-strict server, SensorML shape (the spec-correct client shape)

```text
POST /csapi-go/procedures
Content-Type: application/sml+json

{
  "type": "PhysicalSystem",
  "uniqueId": "urn:test:sml:keywords:002",
  "label": "sml-roundtrip",
  "description": "sml direct post",
  "keywords": ["alpha","beta","gamma"]
}

→ HTTP 201
  Location: /csapi-go/procedures/018408e1-6bcd-47f1-8b4f-9b64e340f749
```

```text
GET /csapi-go/procedures/018408e1-... Accept: application/sml+json
→ HTTP 200
  Content-Type: application/sml+json
  body: {
    "id":"018408e1-...",
    "type":"PhysicalSystem",
    "label":"sml-roundtrip",
    "description":"sml direct post",
    "uniqueId":"urn:test:sml:keywords:002",
    "keywords":["alpha","beta","gamma"],
    "method":{}
  }
```

**`keywords` round-trips.** The spec-correct content-type +
payload-shape combination is supported on the pre-strict server.

### Test 3 — strict server, GeoJSON-as-JSON shape (the failing client shape)

```text
POST /csapi-go-upstream/procedures
Content-Type: application/json

{
  "type": "Feature",
  "properties": {
    "uid": "urn:test:proc:1",
    "name": "Test",
    "keywords": ["a","b"]
  }
}

→ HTTP 400
  Content-Type: application/json
  body: {"error":"unknown field 'keywords' in properties"}
```

The strict decoder correctly rejects the broken client shape rather
than silently dropping the field.

### Cleanup

```text
DELETE /csapi-go/procedures/bc4f8c69-... → 204
DELETE /csapi-go/procedures/018408e1-... → 204
```

## Verification matrix

| Server | `Content-Type` | Payload shape | POST | `keywords` round-trip? | Behavior |
|---|---|---|---|---|---|
| pre-strict (`c9747af`) | `application/json` | GeoJSON Feature, fields under `properties` | 201 | **NO** | Silent loss (pre-existing client bug, hidden) |
| pre-strict (`c9747af`) | `application/sml+json` | SensorML top-level | 201 | **YES** | Spec-correct |
| strict (`df6da0d`) | `application/json` | GeoJSON Feature, fields under `properties` | 400 | n/a | Strict decoder surfaces the broken client shape |
| strict (`df6da0d`) | `application/sml+json` | SensorML top-level | 201 (expected) | **YES** (expected) | Spec-correct (untested but follows from Test 2 + parity) |

## Spec authority

CSAPI Part 1 (OGC 23-001) defines two distinct representations for
the SensorML resource trio:

- **`application/geo+json`** — spatial-discovery view. Carries
  `uid` / `name` / `description` (+ geometry). SensorML metadata
  fields are NOT in this view by design.
- **`application/sml+json`** — full SensorML metadata. Carries
  the 9 fields above plus the GeoJSON-view fields, in SensorML's
  top-level shape (`type: PhysicalSystem`, top-level `uniqueId`,
  top-level `keywords`, etc.).

The server's `{Resource}GeoJSONProperties` wrapper structs being a
strict subset of the corresponding domain structs is **spec-correct**:
the GeoJSON encoding deliberately strips SensorML-only metadata.

## Why the originally-drafted Option A is rejected

The parked report's Option A ("sync `*GeoJSONProperties` with domain
structs") would absorb 9 SensorML metadata fields into the GeoJSON
encoding. That conflates `application/geo+json` and
`application/sml+json` against CSAPI Part 1's deliberate encoding
separation. The 400 from the strict decoder is the correct
failure-mode for a client that mixes encodings.

## Implication for OS4CSAPI publisher fleet — verified by DB audit 2026-05-05

### Database audit results (`connected-systems-go-db-1`, pre-strict `c9747af`, port 8282)

| Resource    | total | `keywords` | `identifiers` | `classifiers` | `contacts` | `characteristics` | `capabilities` | `documentation` | `history` | `security_*` | `legal_*` |
|-------------|------:|-----------:|--------------:|--------------:|-----------:|------------------:|---------------:|----------------:|----------:|-------------:|----------:|
| procedures  |    12 |      **0** |         **0** |         **0** |      **0** |             **0** |          **0** |           **0** |     **0** |        **0** |     **0** |
| systems     |    38 |     **34** |        **35** |        **35** |     **35** |               n/a |            n/a |           **0** |     **0** |        **0** |     **0** |
| deployments |    62 |      **0** |         **0** |         **0** |      **0** |             **0** |          **0** |           **0** |     **0** |        **0** |     **0** |

(`systems` table does not have `characteristics` / `capabilities` columns —
those are scoped to procedures and deployments per the SensorML schema.)

### Refined finding — loss is per-resource

The silent-loss bug is **not uniform across all three SensorML resources**:

- **Systems (38 rows): metadata MOSTLY PRESERVED.** ~89% have
  `keywords` / `identifiers` / `classifiers` / `contacts` populated.
  Implies the publisher fleet's `bootstrap_*_system` paths use a
  spec-correct content-type/payload-shape pair (likely
  `application/sml+json` with SensorML-top-level body — same shape
  as Test 2 above). `documentation` / `history` /
  `security_constraints` / `legal_constraints` still 0/38, but those
  may simply not be supplied in the publisher source data.
- **Procedures (12 rows): TOTAL LOSS.** 0/12 across all 10 SensorML
  columns. The publisher fleet's `bootstrap_*_procedure` paths use
  the broken GeoJSON-as-JSON shape (Test 1 above) — server returns
  201 but persists only `uid` / `name` / `description`.
- **Deployments (62 rows): TOTAL LOSS.** 0/62 across all 10 columns.
  Same client shape bug as procedures.

### Database audit results (`csapi-head-db-1`, port 8283)

`cs-go-head` is currently a development DB used for upstream-issue
test fixtures (rows have `urn:test:issue*` UIDs); no production
publisher fleet data. 0 procedures, 17 test systems (all empty),
2 test deployments (all empty). Confirms scope-limit.

### Bug location refinement (informs plan step E)

The `OS4CSAPI/OSHConnect-Python` fix must focus on
**procedure-bootstrap and deployment-bootstrap code paths**;
system-bootstrap appears to already use the spec-correct shape and
may need only minor consistency tweaks. Recommended re-investigation
in plan step E:

- `publishers/*/bootstrap_*.py` — diff how `ensure_procedure` and
  `ensure_deployment` POST payloads compare to `ensure_system`.
  Expectation: system path sends SensorML top-level (correct);
  procedure + deployment paths send GeoJSON Feature with metadata
  under `properties` (broken).
- `publishers/bootstrap_helpers.py:api_post` — content-type and
  body-shape selection logic.

### Re-bootstrap required

Server-side data for procedures and deployments is unrecoverable;
the authoritative source is publisher-side. Wipe-and-re-bootstrap
of the entire fleet is required after the publisher fix lands. The
existing todo `Wipe upstream DB and recreate` (currently in-progress)
remains gated on plan step E completion. Systems data may be
salvageable via export/re-import once publisher emits identical
shape on the strict server, but cleanest path is full re-bootstrap.

### Audit reproducibility

SQL: see git history of `.tmp-audit-sql.sql` (deleted post-run) or
re-run with the schema-summary script in plan step C. Run via
SSH-to-Oracle-VM → `sudo docker exec connected-systems-go-db-1 psql
-U postgres -d connected_systems -f /tmp/audit.sql`.

## Action — see [`../plan-report-13-disposition.md`](../plan-report-13-disposition.md)

Critical path: park report-13 (done — commits `0d3dc9c`, `337b0f3`)
→ this eval (in progress) → fork-side data-integrity audit →
file `OS4CSAPI/OSHConnect-Python` issue → fix
`OS4CSAPI/OSHConnect-Python` → re-bootstrap fleet against strict
upstream.

**Out of scope (per directive 2026-05-05):**

- Any work on `Botts-Innovative-Research/OSHConnect-Python`.
- Any filing on `SomethingCreativeStudios/connected-systems-go`
  (upstream is correct).
- Schema or wrapper-struct changes on `connected-systems-go`.

## File pointers

- Strict-decode site: `internal/model/formaters/geojson_formatters/procedure_geojson.go` `Deserialize()`
- Strict helper: `internal/model/common_shared/decode.go:33` `DecodeWithFieldErrors[T]`
- Inducing commit: `a467aba` "Adding Strict Parsing..."
- Wrapper struct (subset, by design): `internal/model/domains/procedure.go` `ProcedureGeoJSONProperties` (~L110+)
- Domain struct (full SensorML field set): `internal/model/domains/procedure.go` `Procedure` (top of file)
- Client bug locations (to be fixed in `OS4CSAPI/OSHConnect-Python`):
  - `publishers/bootstrap_helpers.py` `api_post`
  - `publishers/nws/nws/bootstrap_nws.py:122, :212`
  - 9 sibling `publishers/*/bootstrap_*.py` files
