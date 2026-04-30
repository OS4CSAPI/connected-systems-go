# Issue #1 — Datastream creation without explicit `uid` stores empty string, violates unique constraint on second create

- **URL:** <https://github.com/OS4CSAPI/connected-systems-go/issues/1>
- **State at evaluation:** open
- **Labels:** `bug`
- **Filed by:** Sam-Bolling
- **Filed:** 2026-04-17
- **Evaluated against cs-go HEAD:** `1562201a44aa0dbd903ce44a1af40c7f662b2d4a` ("update datastreams", 2026-04-20)
- **Date of evaluation:** 2026-04-30 (initial); revised 2026-04-30 with live empirical evidence from a parallel HEAD deployment (§2.6 update, §2.8 added) and DB-schema introspection of both deployments (§2.6 update).
- **Evaluation methodology:** static code review against HEAD and pre-issue commit `dacae7b`; OGC Part 2 OpenAPI consultation; local `go build` verification; parent-fork cross-check against `SomethingCreativeStudios/connected-systems-go`; live-deployment probe at `https://129-80-248-53.sslip.io/csapi-go/`; **parallel HEAD deployment** at `https://129-80-248-53.sslip.io/csapi-go-head/` for empirical POST testing and DB-schema comparison (raw transcripts: [`docs/research/evidence/issue-001/`](../evidence/issue-001/)).

---

## Sources Consulted

### Primary (authoritative)

- [`internal/model/domains/datastream.go`](../../../internal/model/domains/datastream.go) at HEAD `1562201` — current `Datastream` struct definition.
- [`internal/model/domains/common.go`](../../../internal/model/domains/common.go) at HEAD — `Base` and `CommonSSN` struct definitions, `BeforeCreate` UUID hook.
- [`internal/model/generators/generators_datastream.go`](../../../internal/model/generators/generators_datastream.go) at HEAD — generator that still references `CommonSSN`.
- [`internal/repository/repository.go`](../../../internal/repository/repository.go) at HEAD — `AutoMigrate` call list.
- `git diff dacae7b..1562201 -- internal/model/domains/datastream.go` — shows the structural change after the issue was filed.
- **Parent fork** `SomethingCreativeStudios/connected-systems-go` at HEAD `1562201` (verified via GitHub API 2026-04-30) — same broken `generators_datastream.go`, confirming the defect is in the maintainer's canonical tree, not an OS4CSAPI fork artefact.
- **Live production deployment** at `https://129-80-248-53.sslip.io/csapi-go/` — GET `/datastreams` and `psql \d datastreams` observed 2026-04-30.
- **Parallel HEAD deployment** at `https://129-80-248-53.sslip.io/csapi-go-head/` (cs-go `4b99421`, fresh DB), brought up for this evaluation 2026-04-30 — POST tests T1–T8 and `psql \d datastreams`. Raw transcripts: [`docs/research/evidence/issue-001/head-post-tests-2026-04-30.txt`](../evidence/issue-001/head-post-tests-2026-04-30.txt) and [`docs/research/evidence/issue-001/db-schema-comparison-2026-04-30.txt`](../evidence/issue-001/db-schema-comparison-2026-04-30.txt).
- **OGC API – Connected Systems Part 2, bundled OpenAPI 3.1**, `dataStream` schema: [docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml) lines 266–340 (anchor `&ref_53`) and 1728–1734 (anchor `&ref_54`).
- **OGC API – Connected Systems Part 1, bundled OpenAPI 3.1** — cross-checked for `Datastream` references; spec-mandated fields live in Part 2.
- Local `go build ./...` against HEAD — produced compile error.

### Supporting

- Issue #12 in this repo — corroborates that `unique_identifier` column / index still exists at runtime in deployments.
- [`docs/research/references.md`](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-7/docs/research/references.md) — confirms the two bundled OAS31 YAMLs are the authoritative machine-readable sources for CSAPI; SensorML 3.0 and SWE Common 3.0 JSON Schemas at `schemas.opengis.net` govern SensorML resources and SWE result encoding, not the `dataStream` resource's required-fields list.

---

## 1. Issue body — load-bearing claims

| # | Claim | Source in issue body |
|---|---|---|
| C1 | POSTing a datastream without a `uid` field stores `""` in the database. | "Problem Statement" |
| C2 | The `uid` column has a unique constraint. | "Problem Statement", "Actual behavior" |
| C3 | A second create without `uid` fails with a PostgreSQL unique constraint violation referencing a constraint name. | "Actual behavior" |
| C4 | The `id` field is auto-generated server-side, providing precedent for auto-generating `uid`. | "Option A — Recommended" |
| C5 (implicit) | OGC 23-002 §9.2 mandates a `uid` field on Datastream resources. | Implied by ownership = "connected-systems-go" and reference to "OGC 23-002 §9.2 Datastream resource definition" |

Stated severity: P1-Critical · Stated category: API Design / Data Integrity · Stated ownership: connected-systems-go.

---

## 2. Verification

### 2.1 — `Datastream` had a `uid` field with a uniqueIndex at filing time (supports C1, C2)

**Evidence (pre-fix code at commit `dacae7b`, reconstructed from `git diff dacae7b..1562201`):**

```diff
 type Datastream struct {
     Base
-    CommonSSN
+    Name        string `gorm:"type:varchar(255);not null" json:"name"`
+    Description string `gorm:"type:text" json:"description,omitempty"`
```

**Evidence (`CommonSSN` definition, [common.go](../../../internal/model/domains/common.go) lines 16–22, unchanged at HEAD):**

```go
type CommonSSN struct {
    UniqueIdentifier UniqueID `gorm:"type:varchar(255);uniqueIndex" json:"uid"`
    Name             string   `gorm:"type:varchar(255);not null" json:"name"`
    Description      string   `gorm:"type:text" json:"description,omitempty"`
}
```

**Analysis:** At filing time, `Datastream` embedded `CommonSSN`. The `UniqueIdentifier` field is a `UniqueID` (alias for `string`), with no `omitempty`, no pointer indirection, and no auto-generation hook. Go's zero value for `string` is `""`. JSON unmarshalling a body without a `uid` key leaves `UniqueIdentifier == ""`. GORM's `uniqueIndex` directive emits a unique index on the `unique_identifier` column. PostgreSQL unique indexes treat `''` as a value, not NULL, so two inserts of `''` collide. C1 and C2 are confirmed structurally; C3 follows as a logical consequence.

### 2.2 — `id` precedent (supports C4)

**Evidence ([common.go](../../../internal/model/domains/common.go) lines 25–31):**

```go
func (b *Base) BeforeCreate(tx *gorm.DB) error {
    if b.ID == "" {
        b.ID = uuid.New().String()
    }
    return nil
}
```

**Analysis:** `Base.ID` has a `BeforeCreate` GORM hook that auto-generates a UUID when omitted. `UniqueIdentifier` has no equivalent hook. C4 is accurate.

### 2.3 — OGC Part 2 spec does NOT mandate a `uid` field on `dataStream` (refutes C5)

**Evidence (Part 2 bundled OpenAPI, lines 266–315 — `&ref_53` anchor, the `dataStream`'s `allOf` definition):**

```yaml
allOf: &ref_53
  - $schema: https://json-schema.org/draft/2020-12/schema
    type: object
    properties: &ref_79
      id:
        description: Local resource ID. If set on creation, the server may ignore it.
        type: string
        minLength: 1
        readOnly: true
      name:
        description: Human readable name of the resource
        type: string
        minLength: 1
      description:
        description: Human readable description of the resource
        type: string
        minLength: 1
      validTime: ...
      formats: ...
    required: &ref_80
      - id
      - name
      - formats
```

**Evidence (Part 2 bundled OpenAPI, lines 1728–1734 — `&ref_54` anchor, the `dataStream`'s top-level `required` list):**

```yaml
required: &ref_54
  - name
  - system@link
  - observedProperties
  - phenomenonTime
  - resultTime
  - resultType
  - live
```

**Analysis:** The OGC 23-002 `dataStream` schema does not define a `uid` property at all. The required-fields list contains seven entries, none of which is `uid`. The `uid` references found elsewhere in the spec (e.g. line 355) are properties of the `Link` object schema (a target-resource UID inside a hyperlink), not of `dataStream` itself. **C5 is refuted: cs-go's `uid`/`unique_identifier` field on Datastream is a server-side invention, not an OGC requirement.** This reframes the issue: the discussion is no longer "how should cs-go satisfy the spec's `uid` requirement?" but "should cs-go expose a `uid` at all, given the spec doesn't define one?"

### 2.4 — Code state at HEAD removed `CommonSSN` from `Datastream` (silent partial fix)

**Evidence ([datastream.go](../../../internal/model/domains/datastream.go) at HEAD lines 14–18):**

```go
type Datastream struct {
    Base
    Name        string `gorm:"type:varchar(255);not null" json:"name"`
    Description string `gorm:"type:text" json:"description,omitempty"`
    ...
}
```

**Evidence (`git log -1 --format="%s" 1562201`):**

```
update datastreams
```

**Analysis:** Commit `1562201` (2026-04-20, three days after the issue was filed) removed `CommonSSN` from `Datastream` and replaced it with explicit `Name` and `Description` fields. This silently eliminates the `uid` field from the struct — but the commit message says only "update datastreams", does not reference the issue, and the issue was not closed. **None of the issue's three proposed options were chosen; a fourth path ("remove the field entirely") was taken without discussion.** Notably, "remove the field" aligns with §2.3's spec finding — the field was not OGC-mandated — so the direction of this change is defensible even though the process around it was not.

### 2.5 — The "fix" leaves the test-fixtures package broken (active defect, scope: tests only)

**Evidence ([generators_datastream.go](../../../internal/model/generators/generators_datastream.go) lines 175–181 at HEAD):**

```go
return domains.Datastream{
    Base: domains.Base{ID: id},
    CommonSSN: domains.CommonSSN{
        UniqueIdentifier: domains.UniqueID(fmt.Sprintf("urn:uuid:%s", id)),
        Name:             "Datastream " + f.Lorem().Word(),
        Description:      f.Lorem().Sentence(6),
    },
    ...
}
```

**Evidence (build matrix at HEAD `1562201`, run 2026-04-30):**

```
$ go build ./cmd/server      # deployable server binary
EXIT=0                       # builds cleanly

$ go build ./...             # whole module
# github.com/yourusername/connected-systems-go/internal/model/generators
internal\model\generators\generators_datastream.go:177:3: unknown field CommonSSN in struct literal of type domains.Datastream
EXIT=1

$ go test -count=0 ./...     # compile-only test pass
# (same error)
EXIT=1
```

**Evidence (parent fork `SomethingCreativeStudios/connected-systems-go` HEAD `1562201`, fetched via GitHub API 2026-04-30, lines around 175):**

```go
return domains.Datastream{
        Base: domains.Base{ID: id},
        CommonSSN: domains.CommonSSN{
                UniqueIdentifier: domains.UniqueID(fmt.Sprintf("urn:uuid:%s", id)),
                Name:             "Datastream " + f.Lorem().Word(),
                Description:      f.Lorem().Sentence(6),
        },
        ...
}
```

**Evidence (use-site for the broken package; `package generators` is imported only from `_test.go` files):**

```
$ Get-ChildItem -Recurse -Filter *.go | Select-String 'model/generators' | Where-Object { $_.Path -notmatch '_test\.go$' }
# (no output — no non-test importer)
```

**Analysis:** The defect is real but narrower than "the project doesn't build":

- **Deployable artefact (`./cmd/server`) builds cleanly.** Anyone running CI as `go build ./cmd/server` to produce the release binary sees no break. This is consistent with a working production deployment.
- **Module-wide build (`go build ./...`) fails** because `internal/model/generators/` constructs `domains.Datastream{ CommonSSN: ... }` against a struct that no longer has that embedded field after `1562201`. That package is a test-fixture / fake-data helper used only by `_test.go` files (no non-test importer).
- **`go test ./...` consequently fails to compile** at HEAD on both OS4CSAPI and the maintainer's parent fork (verified above), so the issue is upstream and not an OS4CSAPI-fork artefact.

The practical impact: production deployments are unaffected; the test suite is unrunnable until the fakes are updated. CI that gates on `go test ./...` would catch this; CI that only builds `./cmd/server` would not.

### 2.6 — DB column / unique-index drift (active defect)

**Evidence ([repository.go](../../../internal/repository/repository.go) lines 47–69):**

```go
func AutoMigrate(db *gorm.DB) error {
    ...
    if err := db.AutoMigrate(
        &domains.System{},
        &domains.Deployment{},
        ...
        &domains.Datastream{},
        ...
    ); err != nil {
        return err
    }
    ...
}
```

**Evidence (issue #12 title):**

> "Datastream `unique_identifier` enforces global uniqueness — should be scoped per parent system"

**Analysis:** GORM's `AutoMigrate` adds columns and indexes but **does not drop columns or indexes when a Go field is removed from a struct.** Any cs-go deployment whose database was initialized before `1562201` retains the `unique_identifier` column with its unique index. Issue #12, filed after `1562201`, presupposes the column still exists in deployment, confirming this drift in practice.

**Live DB evidence (`psql \d datastreams` against both deployments, 2026-04-30):**

Production DB (pre-`1562201` code, instance running on `localhost:8282`):

```
 unique_identifier        | character varying(255)   |           |          |
 ...
Indexes:
    "datastreams_pkey" PRIMARY KEY, btree (id)
    "idx_datastreams_unique_identifier" UNIQUE, btree (unique_identifier)
    ...
```

Fresh DB initialized by HEAD `4b99421` code (this evaluation's parallel deploy on `localhost:8283`):

```
(no unique_identifier column; no idx_datastreams_unique_identifier index)
```

This is direct evidence for both branches of the drift hypothesis:

- **Fresh DB initialized post-`1562201`:** Confirmed — `unique_identifier` is not added (the field is gone from the struct), and the unique index does not exist. The original C1/C2/C3 collision path is gone for new installs, but the `uid` field is also silently absent from the API surface.
- **DB initialized pre-`1562201`:** Confirmed — the column persists, the `UNIQUE` index persists, and the column is nullable (no `NOT NULL`). If HEAD code were upgraded onto this DB without a migration, the column would remain populated for legacy rows but new rows from HEAD code would have it `NULL`; Postgres permits multiple `NULL`s in a UNIQUE index, so the legacy unique constraint would not collide with HEAD inserts. The original C1 empty-string collision behavior is, however, still latently reachable through the running pre-fix code path against this DB.

### 2.7 — Live deployment is running pre-`1562201` code (corroborates partial-fix finding)

**Evidence (GET `https://129-80-248-53.sslip.io/csapi-go/datastreams`, 2026-04-30, first record):**

```json
{
  "id":"120b633b-aff8-4710-810f-ce4b25012676",
  "uid":"urn:os4csapi:datastream:usgs-eq-feed:earthquakeEvent:v1",
  "name":"Earthquake Events",
  "description":"Normalized earthquake events from the USGS GeoJSON summary feed…",
  "system@link":{"href":"systems/8eee605f-0751-4c50-ac97-46d74ab0309d"},
  "outputName":"earthquakeEvent",
  "schema":{…},
  "links":[…],
  "Systems":null
}
```

**Evidence (current `Datastream` struct at HEAD `1562201`, [datastream.go](../../../internal/model/domains/datastream.go) lines 14–18):**

```go
type Datastream struct {
    Base
    Name        string `gorm:"type:varchar(255);not null" json:"name"`
    Description string `gorm:"type:text" json:"description,omitempty"`
    ...
}
```

**Analysis:** The live deployment serves datastreams with a populated `uid` field. The current struct at HEAD has **no `uid` field** — there is no JSON tag that could produce one in the response. The deployed binary therefore cannot have been built from `1562201`; it is from an earlier commit (most plausibly `dacae7b` "Bug fixes and clean up" or earlier, where `Datastream` still embedded `CommonSSN`). This is direct external evidence that:

1. Whatever process built `1562201` did not roll forward to the deployment, so the partial-fix hasn't reached production.
2. The original bug C1–C3 (empty-string `uid` collision) **is still reachable on the live deployment** if a client POSTs a datastream without `uid`. Worth a follow-up reproduction once authentication is established — but the existing populated `uid` values shown above are well-formed URNs, suggesting the publishers are correctly setting `uid` and the empty-string code path may simply not have been exercised yet in this deployment.
3. The deployment's database almost certainly has the `unique_identifier` column with its unique index (per §2.6, scenario "pre-`1562201`"), so when the next deploy ships `1562201` the column will remain in the DB even though the API stops emitting it — the worst of both worlds, until a migration is added.

*(Update 2026-04-30, parallel deploy:)* §2.6's live `psql` snapshot of the production DB now confirms the third bullet directly: `unique_identifier` does exist with its `UNIQUE` index in the running production database.

### 2.8 — Live HEAD empirical verification (parallel deploy)

**Deployment under test:** A second cs-go stack was brought up on the same Oracle host on 2026-04-30, built from this fork's `main` at commit `4b99421` (which is `1562201` plus only documentation commits for this evaluation). Project name `csapi-head`, port `8283`, separate Postgres volume `postgres_data_head`, base URL `https://129-80-248-53.sslip.io/csapi-go-head`. Caddy route added at `/csapi-go-head/*`. The fresh database was initialized by HEAD code's `AutoMigrate` (see §2.6).

**Test sequence (no auth on this route; full `curl -isS` transcripts at [`docs/research/evidence/issue-001/`](../evidence/issue-001/)):**

| Step | Request | Response |
|---|---|---|
| T1 | `GET /datastreams` | `200 OK` — `{"items":[],"links":[…]}` |
| T2 | `GET /api` | `200 OK` — body is **86 bytes**: `{"openapi":"3.0.0","info":{"title":"OGC Connected Systems API","version":"1.0.0"}}`. No `paths`, no `components.schemas`. The `/api` endpoint at HEAD is a stub. |
| T3 | `POST /systems` (well-formed Feature) | `201 Created` |
| T4 | `POST /systems/{sysId}/datastreams` with body **omitting `uid`** | `201 Created`, `Location: …/datastreams/cf4cd80c-…` |
| T5 | Same body, **second create** | `201 Created`, `Location: …/datastreams/e4edced2-…` (fresh UUID, no error) |
| T6 | Same shape but with stray top-level `"uid":"urn:test:issue1:ds:stray"` | `201 Created`, `Location: …/datastreams/186b7853-…` |
| T7 | `GET /datastreams` after T4–T6 | `200 OK`, three items returned. **Every item's keys:** `[Systems, description, id, links, live, name, observedProperties, outputName, resultType, system@link]`. **No `uid` key** on any record, including the one created in T6 with a stray `uid` in the request body. |

**Note on the canonical create path:** `POST /datastreams` at both deployments returns `405 Method Not Allowed` with `Allow: GET`. Datastream creation is registered only under `/systems/{id}/datastreams` ([`internal/api/router.go`](../../../internal/api/router.go) line 126). Initial probes at `POST /datastreams` were rejected for this reason, not for any reason related to `uid`.

**Analysis:**

1. **C1 (empty-string `uid` stored on POST without `uid`) is not reachable on HEAD.** T4 and T5 both succeed; the response shape contains no `uid` and the DB has no `unique_identifier` column to collide on. This refutes C1 *as a current symptom on HEAD* and confirms that `1562201` (the silent partial fix) successfully eliminates the immediate symptom for fresh installs. C1 remains a true historical statement about pre-fix code (§2.1), and its underlying empty-string code path is still latently present in the production deployment that runs pre-`1562201` code (§2.7).
2. **C2 / C3 (unique-constraint violation on second create) are not reachable on HEAD.** T5 returns `201 Created`. The column and index do not exist (§2.6 HEAD snapshot).
3. **Stray `uid` in request body is silently dropped.** T6 succeeds and the resulting record (visible in T7) has no `uid`. This is a separate, smaller defect introduced by the silent partial-fix: a client written against a pre-`1562201` API who continues to send `uid` will receive `201 Created` with no warning that the field has been discarded. The `Datastream` struct uses `json:` tags that the standard library `encoding/json` decoder ignores unknown fields by default, so this is a silent-data-loss surface, not a hard-error surface. Worth a separate follow-up issue.
4. **Response shape includes `Systems` (capital S).** T7 shows every record has a top-level `"Systems":null` key. This is the GORM relationship field name leaking into JSON output (no `json:"-"` tag). Not a defect of issue #1, but a surface-quality finding the issue body did not anticipate.
5. **`/api` is a stub at HEAD.** T2's 86-byte response means the OpenAPI document advertised by the landing page does not actually describe any paths or schemas. A client cannot programmatically discover whether `uid` is part of the data model from `/api`; this raises the bar for the "silent removal is acceptable" argument because there is no machine-readable contract for clients to compare against. Separate follow-up issue territory.

---

## 3. Reproduction

**Static reproduction:** Confirmed via build matrix (§2.5): `go build ./cmd/server` succeeds (EXIT=0); `go build ./...` and `go test -count=0 ./...` fail with the `CommonSSN` error in `internal/model/generators/`. The deployable artefact is unaffected; the test fixtures package is broken.

**Cross-fork verification:** Parent fork `SomethingCreativeStudios/connected-systems-go` HEAD `1562201` carries the identical broken file. Compile break is upstream, not on our end (§2.5).

**Live deployment probe (production, pre-`1562201`):** GET `/` and GET `/datastreams` against `https://129-80-248-53.sslip.io/csapi-go/` succeeded (HTTP 200). Response confirms the deployed build is pre-`1562201` (§2.7). A direct reproduction of the original C1–C3 empty-string collision was not attempted on production because it would mutate the live publisher dataset; the symptom is, however, latently reachable per §2.6 and §2.7.

**Live deployment probe (HEAD `4b99421`, parallel deploy):** As recorded in §2.8, three POST attempts against `/systems/{id}/datastreams` — one without `uid`, one repeating the same body, one with a stray `uid` — all returned `201 Created`. None of the resulting records carry a `uid` field. The original C1/C2/C3 symptoms are not reproducible on HEAD; the silent-partial-fix has eliminated them for fresh installs.

---

## 4. Verdicts

| Dimension | Verdict | Rationale (one sentence) |
|---|---|---|
| Validity | `partially-confirmed` | Original bug logically confirmed at filing time; not directly reproducible at HEAD because field was removed; runtime situation unresolved due to DB drift, but live deployment still runs pre-fix code so original symptom likely still reachable in production. |
| Legitimacy | `defect` | The pre-fix combination of `uniqueIndex` + no `omitempty` + no auto-generation is internally inconsistent independent of any spec opinion. |
| Accuracy | `accurate-but-stale` | Every cited C1–C4 claim verifies against pre-fix code. C5 (implicit spec mandate) is refuted by the OGC OpenAPI. The issue body is now stale because of the silent partial fix. |
| Completeness | `complete` | All standard sections present; could not have anticipated `1562201`. |

**Overall recommendation: Keep with edits.**

---

## 5. Reasoning summary

The issue is well-written and accurately described a real defect at the time it was filed (§2.1). However, three days later, commit `1562201` made a silent structural change that eliminates the immediate symptom by removing the field entirely (§2.4) — a path none of the issue's three proposed options recommended. That change is **directionally defensible** because §2.3 establishes that OGC 23-002 does not mandate a `uid` on Datastream in the first place (refuting load-bearing claim C5).

The current state has loose ends, though smaller than first appearances suggested: (a) the **test-fixtures package** doesn't compile (§2.5) — the deployable server binary is fine, but `go test ./...` is broken on both forks; (b) deployed databases retain the column and unique index, **now confirmed live via psql** (§2.6); (c) issue #12 exists separately and presupposes the column is still present; (d) the live production deployment is still running pre-`1562201` code (§2.7) so the original empty-string collision is latently reachable; (e) at HEAD the silent partial fix successfully eliminates the C1–C3 symptoms (§2.8 T4–T7: three POST variants all return `201 Created`, no `uid` in responses, no unique-constraint violation), but introduces three smaller defects: silent drop of stray `uid` in request bodies, `/api` is a stub, and the GORM relationship field `Systems` leaks into JSON output; and (f) the issue tracker has no record of this trajectory because the commit message did not reference the issue and the issue was not closed.

The retroactive picture is: there is no spec-mandated `uid` field on Datastream; cs-go's original surface was an over-specification; the silent removal is the right direction but the execution left the test suite broken, the deployments out of sync, and the issue tracker silent on the trajectory.

---

## 6. Recommendation

**Keep with edits.** Suggested edits to the issue body, in priority order:

1. **Add a "Status update (2026-04-30)" section** stating that commit `1562201` removed `CommonSSN` from the `Datastream` struct, was not announced as a fix for this issue, broke the test-fixtures package (`go test ./...` does not compile, though `./cmd/server` does), and may not have updated existing databases.
2. **Add a "Spec basis" note**: per the OGC 23-002 Part 2 bundled OpenAPI (lines 266–340, 1728–1734), Datastream has no `uid` property. The original three options (auto-generate / nullable / require) all assumed cs-go must keep the field; per the spec, removing it is also valid and is what `1562201` did.
3. **Cross-reference #12** as a sibling tracking the surviving DB-column / index aspect.
4. **Either close as superseded** by `1562201` plus a follow-up issue for the build break and DB-migration plan, **or** keep open as the canonical tracker for finishing the cleanup.

Do **not** silently close as fixed: the partial-fix state is real and visible (test-suite break, DB drift, deployment running older code), and the trail of decisions deserves to remain auditable.

---

## 7. Open questions / unknowns

1. **Author intent for `1562201`.** Was it intended as a fix for this issue? The commit message does not say. Asking the author would resolve this. **Still open.**
2. **Live DB schemas.** **Resolved 2026-04-30** via parallel-deploy psql introspection (§2.6, §2.8). Production DB has `unique_identifier varchar(255)` (nullable, no `NOT NULL`) with `idx_datastreams_unique_identifier UNIQUE`. HEAD-initialized DB does not have the column or index.
3. **`?uid=` query parameter (issue #7).** With no `uid` field on Datastream at all in the *code* but `uid` still emitted by the *deployed* API, `?uid=` filtering may behave inconsistently across deployments depending on which build is running. Defer to issue #7's evaluation. **Still open** — will be addressed when issue #7 is evaluated; the parallel deploy now provides a clean test bed for it.
4. **Migration plan.** If "remove `uid`" is the chosen direction, what migration ships with the change to drop the column and index from existing databases? Not currently in any commit on `main`, in either fork. **Still open.**
5. **Deployment build provenance.** Which exact commit produced the binary running at `129-80-248-53.sslip.io/csapi-go/`? The response shape rules out `1562201`; it is likely `dacae7b` or earlier but a commit-tagged build artefact or a `/about` endpoint would make this verifiable. **Still open** (response shape narrows it to commits where `Datastream` still embeds `CommonSSN`, but does not pin to a single commit).
6. **Stray-`uid` silent-drop behavior (new, from §2.8 T6).** Should HEAD reject requests that include a `uid` field with `400 Bad Request`, or document the silent-drop, or accept-and-ignore as it does today? Current behavior is silent data loss with `201 Created`. **Worth a separate follow-up issue.**
7. **`/api` stub at HEAD (new, from §2.8 T2).** The OpenAPI document advertised by the landing page returns 86 bytes with no paths or schemas. Is the bundled OpenAPI from this fork (under `docs/research/standards/`) intended to be served, or is `/api` a deliberate placeholder? **Worth a separate follow-up issue.**
8. **Capital-S `Systems` field leaking into JSON responses (new, from §2.8 T7).** Each datastream record returns a top-level `"Systems":null`. The struct field is the GORM `has-many` relationship without a `json:"-"` tag. Not part of issue #1, but a surface-quality finding worth filing separately.
