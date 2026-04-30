# Issue #1 — Datastream creation without explicit `uid` stores empty string, violates unique constraint on second create

- **URL:** <https://github.com/OS4CSAPI/connected-systems-go/issues/1>
- **State at evaluation:** open
- **Labels:** `bug`
- **Filed by:** Sam-Bolling
- **Filed:** 2026-04-17
- **Evaluated against cs-go HEAD:** `1562201a44aa0dbd903ce44a1af40c7f662b2d4a` ("update datastreams", 2026-04-20)
- **Date of evaluation:** 2026-04-30
- **Evaluation methodology:** static code review against HEAD and pre-issue commit `dacae7b`; OGC Part 2 OpenAPI consultation; local `go build` verification; parent-fork cross-check against `SomethingCreativeStudios/connected-systems-go`; live-deployment probe at `https://129-80-248-53.sslip.io/csapi-go/`. No authenticated POST attempted against the deployment.

---

## Sources Consulted

### Primary (authoritative)

- [`internal/model/domains/datastream.go`](../../../internal/model/domains/datastream.go) at HEAD `1562201` — current `Datastream` struct definition.
- [`internal/model/domains/common.go`](../../../internal/model/domains/common.go) at HEAD — `Base` and `CommonSSN` struct definitions, `BeforeCreate` UUID hook.
- [`internal/model/generators/generators_datastream.go`](../../../internal/model/generators/generators_datastream.go) at HEAD — generator that still references `CommonSSN`.
- [`internal/repository/repository.go`](../../../internal/repository/repository.go) at HEAD — `AutoMigrate` call list.
- `git diff dacae7b..1562201 -- internal/model/domains/datastream.go` — shows the structural change after the issue was filed.
- **Parent fork** `SomethingCreativeStudios/connected-systems-go` at HEAD `1562201` (verified via GitHub API 2026-04-30) — same broken `generators_datastream.go`, confirming the defect is in the maintainer's canonical tree, not an OS4CSAPI fork artefact.
- **Live deployment** at `https://129-80-248-53.sslip.io/csapi-go/` — GET `/datastreams` response observed 2026-04-30.
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

### 2.5 — The "fix" leaves the build broken (active defect; verified in parent fork too)

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

**Evidence (`go build ./...` at HEAD `1562201`):**

```
# github.com/yourusername/connected-systems-go/internal/model/generators
internal\model\generators\generators_datastream.go:177:3: unknown field CommonSSN in struct literal of type domains.Datastream
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

**Analysis:** The compile break is not an artefact of the OS4CSAPI fork or local clone state — the parent fork (`SomethingCreativeStudios/connected-systems-go`) at the same HEAD `1562201` ships identical broken code. `generators_datastream.go` constructs `domains.Datastream` with a `CommonSSN: domains.CommonSSN{...}` literal that is no longer valid against the current `Datastream` definition. The file has no build tag and is imported by `e2e/observations_test.go:15`. Both forks' `main` branches do not compile. CI signal on either repository is therefore not what it appears.

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

**Analysis:** GORM's `AutoMigrate` adds columns and indexes but **does not drop columns or indexes when a Go field is removed from a struct.** Any cs-go deployment whose database was initialized before `1562201` retains the `unique_identifier` column with its unique index. Issue #12, filed after `1562201`, presupposes the column still exists in deployment, confirming this drift in practice. The runtime situation depends on which DB state a server is running against:

- **Fresh DB initialized post-`1562201`:** No `unique_identifier` column added (struct has no field). The original C1/C2/C3 bug is gone, but `uid` is silently absent from the API surface.
- **DB initialized pre-`1562201`:** Column persists with `uniqueIndex`. Inserts from current code do not populate it, so behavior depends on the column's NOT NULL / DEFAULT constraints, which I did not verify against a live DB.

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

---

## 3. Reproduction

**Static reproduction:** Confirmed via `go build ./...` (§2.5) — current `main` does not compile.

**Cross-fork verification:** Parent fork `SomethingCreativeStudios/connected-systems-go` HEAD `1562201` carries the identical broken file. Compile break is upstream, not on our end (§2.5).

**Live deployment probe:** GET `/` and GET `/datastreams` against `https://129-80-248-53.sslip.io/csapi-go/` succeeded (HTTP 200). Response confirms the deployed build is pre-`1562201` (§2.7). A direct reproduction of the original C1–C3 empty-string collision was not attempted because (a) it requires authenticated POST against the deployment which exceeds passive evaluation scope, and (b) the deployed code path predates the fix attempt anyway, so a successful reproduction would only re-confirm what §2.1 already establishes.

---

## 4. Verdicts

| Dimension | Verdict | Rationale (one sentence) |
|---|---|---|
| Validity | `partially-confirmed` | Original bug logically confirmed at filing time; not directly reproducible at HEAD because field was removed; runtime situation unresolved due to DB drift. |
| Legitimacy | `defect` | The pre-fix combination of `uniqueIndex` + no `omitempty` + no auto-generation is internally inconsistent independent of any spec opinion. |
| Accuracy | `accurate-but-stale` | Every cited C1–C4 claim verifies against pre-fix code. C5 (implicit spec mandate) is refuted by the OGC OpenAPI. The issue body is now stale because of the silent partial fix. |
| Completeness | `complete` | All standard sections present; could not have anticipated `1562201`. |

**Overall recommendation: Keep with edits.**

---

## 5. Reasoning summary

The issue is well-written and accurately described a real defect at the time it was filed (§2.1). However, three days later, commit `1562201` made a silent structural change that eliminates the immediate symptom by removing the field entirely (§2.4) — a path none of the issue's three proposed options recommended. That change is **directionally defensible** because §2.3 establishes that OGC 23-002 does not mandate a `uid` on Datastream in the first place (refuting load-bearing claim C5).

The current state is, however, worse than either fully fixing or fully reverting: (a) the build is broken (§2.5), (b) deployed databases retain the column and unique index (§2.6), (c) issue #12 exists separately and presupposes the column is still present, and (d) the issue tracker has no record of this trajectory because the commit message did not reference the issue and the issue was not closed.

The retroactive picture is: there is no spec-mandated `uid` field on Datastream; cs-go's original surface was an over-specification; the silent removal is the right direction but the execution left the codebase broken and the deployments out of sync.

---

## 6. Recommendation

**Keep with edits.** Suggested edits to the issue body, in priority order:

1. **Add a "Status update (2026-04-30)" section** stating that commit `1562201` removed `CommonSSN` from the `Datastream` struct, was not announced as a fix for this issue, broke the `go build`, and may not have updated existing databases.
2. **Add a "Spec basis" note**: per the OGC 23-002 Part 2 bundled OpenAPI (lines 266–340, 1728–1734), Datastream has no `uid` property. The original three options (auto-generate / nullable / require) all assumed cs-go must keep the field; per the spec, removing it is also valid and is what `1562201` did.
3. **Cross-reference #12** as a sibling tracking the surviving DB-column / index aspect.
4. **Either close as superseded** by `1562201` plus a follow-up issue for the build break and DB-migration plan, **or** keep open as the canonical tracker for finishing the cleanup.

Do **not** silently close as fixed: the partial-fix state is real and visible (build break, DB drift), and the trail of decisions deserves to remain auditable.

---

## 7. Open questions / unknowns

1. **Author intent for `1562201`.** Was it intended as a fix for this issue? The commit message does not say. Asking the author would resolve this.
2. **Live DB schemas.** Per §2.7 the deployed binary is pre-`1562201`, so the `unique_identifier` column almost certainly exists in that DB. Direct DDL introspection (e.g. via a status endpoint or DB access) would confirm exact column NULL/UNIQUE state and whether the live empty-string collision is still reachable.
3. **`?uid=` query parameter (issue #7).** With no `uid` field on Datastream at all in the *code* but `uid` still emitted by the *deployed* API, `?uid=` filtering may behave inconsistently across deployments depending on which build is running. Defer to issue #7's evaluation.
4. **Migration plan.** If "remove `uid`" is the chosen direction, what migration ships with the change to drop the column and index from existing databases? Not currently in any commit on `main`, in either fork.
5. **Deployment build provenance.** Which exact commit produced the binary running at `129-80-248-53.sslip.io`? The response shape rules out `1562201`; it is likely `dacae7b` or earlier but a commit-tagged build artefact or a `/about` endpoint would make this verifiable.
