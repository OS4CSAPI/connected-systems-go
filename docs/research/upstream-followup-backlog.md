# Upstream Follow-up Backlog

Tracks residual defects, enhancements, and unscoped questions surfaced during the
26-issue closure pass on `OS4CSAPI/connected-systems-go`, to be filed as fresh
issues on `SomethingCreativeStudios/connected-systems-go` once the closure pass
on this fork is complete.

Each entry: source issue → category (defect / enhancement / question) → file:line
anchors → one-line summary → status.

---

## Open items

### 1. #9 — pagination default UX
- **Source:** Issue #9 (closed-as-completed via `635547f`).
- **Category:** Resolved upstream.
- **Summary:** Maintainer adopted Option B from the issue body —
  configurable via `api.default_limit` (default kept at spec-conformant
  `10`). No follow-up needed.
- **Status:** **CLOSED** — no upstream issue to file. Comment 4380531930.

### 2. #10 — remaining `documentation` → `documents` sites
- **Source:** Issue #10 (left open with status comment 4379050259).
- **Category:** Defect (P2).
- **Summary:** `c2ab201` fixed only 1 of 9 sites. 7 remain on `upstream/main`,
  including the input-side struct `internal/model/domains/system.go:41`, which
  means the headline POST symptom still reproduces.
- **Status:** Holding for maintainer scope reply on #10. If maintainer says
  "fix the rest", we keep #10 open and PR. If "out of scope", file as new
  issue with the full 7-site list.

### 3. #14 — `/api` endpoint stub
- **Source:** Issue #14 (closed-as-not-planned on fork, comment 4380617500).
- **Category:** Defect (P2 — binding spec `SHALL` violation).
- **Summary:** `GET /api` returns an 88-byte JSON literal with only
  `openapi` + `info`. No `paths`, no `components`, no `servers`.
  `internal/api/router.go` `getOpenAPISpec` still has the
  `// TODO: Implement OpenAPI 3.0 spec generation` comment. Verified
  unchanged on `upstream/main` HEAD `2dc09f7`. Violates OAS 3.0.3
  §4.7.1.1 (`paths` REQUIRED) and OGC 19-072
  `/req/landing-page/api-definition-success` clauses B + C
  (server self-declares `conf/landing-page`).
- **Recommendation:** Option 3 (serve upstream OGC OAS bundles with
  patched `servers`, shift media type to `version=3.1`) → fallback
  Option 1 (`//go:embed` curated bundle).
- **Status:** Ready to file against
  `SomethingCreativeStudios/connected-systems-go` after closure pass.
  Reference our eval + evidence files so maintainer has full chain.

### 4. Datastream `applyFilters` dangling `unique_identifier` SQL
- **Source:** Issue #12 closure (finding A).
- **Category:** Defect (P3 — latent / unreachable but wrong).
- **Summary:** `internal/repository/datastream_repository.go` `applyFilters`
  still contains a `unique_identifier` filter clause on a column that was
  removed from the Datastream domain model. Currently unreachable (no handler
  wires it), but will silently fail at runtime if any code path adds it back.
- **Status:** Ready to file after closure pass.

### 5. ControlStream `Systems` field JSON leak
- **Source:** Issue #15 closure (§3.4).
- **Category:** Defect (**P3-Minor** — sibling of fix that landed on Datastream;
  severity corrected from initial "P2" to match parent issue #15's accepted
  triage. Sibling cannot exceed parent severity per eval §3.6.).
- **Summary:** `2dc09f7` added `json:"-"` to `Datastream.Systems` to prevent
  the GORM many2many slice from serializing into API responses. The exact
  same field exists on `ControlStream` and was not given the same tag, so
  `GET /controlstreams/{id}` still leaks the join data as `"Systems": null`
  at top level.
- **Reported in:** [`upstream-issue-reports/report-03-controlstream-systems-json-leak.md`](upstream-issue-reports/report-03-controlstream-systems-json-leak.md).
- **Status:** Ready to file after closure pass. One-line fix.

### 6. Latent untagged relationship slices (optional hardening)
- **Source:** Issue #15 closure (§3.5).
- **Category:** Enhancement.
- **Summary:** Multiple `[]Foo` slices on domain structs in
  `internal/model/domains/system.go` and `procedure.go` lack any GORM
  relationship tags. AutoMigrate ignores them silently, so they appear in API
  responses as empty arrays even when relations exist. Optional; not
  user-facing today.
- **Status:** File only if maintainer wants schema-side hardening.

### 7. `SystemEvent` not handled in `SystemRepository.deleteCascade`
- **Source:** Issue #16 closure.
- **Category:** Defect (P2 — orphaning).
- **Summary:** `fe9fbd0` chose Approach 3b (application-level child checks)
  over Approach 3a (DB-level FK constraints). Under 3b, every child resource
  must be enumerated in `deleteCascade`. `SystemEvent` is not. Deleting a
  system with attached events will succeed and orphan the events rows.
- **Status:** Ready to file after closure pass. Small addition to
  `system_repository.go` `deleteCascade`'s child list.

### 8. Malformed UUID path-param → 400 (currently 500) — **CLOSED, NOT A DEFECT**
- **Source:** Issue #17 closure.
- **Category:** ~~Defect (P3 — UX / contract).~~ **Withdrawn 2026-05-05.**
- **Original summary:** `DELETE /datastreams/not-a-uuid` and similar were
  expected to produce SQLSTATE `22P02` (`invalid_text_representation`) at
  the DB and fall through to a default-500 arm.
- **Re-verification finding (2026-05-05, `upstream/main` HEAD `df6da0d`):**
  The premise is wrong. CSAPI domain types declare their primary key as
  `ID string gorm:"primaryKey;type:varchar(255)"` (see
  `internal/model/domains/common.go:12` and `collection.go:9`), **not**
  PostgreSQL `uuid` type. A non-UUID path parameter is therefore a valid
  `varchar(255)` value that simply doesn't match any row; PostgreSQL
  never raises `22P02`. Live test against `csapi-go-upstream` (`df6da0d`)
  confirms all four observable shapes already return **HTTP 404
  ErrNotFound**, not 500:
  ```text
  DELETE /datastreams/not-a-uuid     HTTP 404
  GET    /datastreams/not-a-uuid     HTTP 404
  DELETE /systems/not-a-uuid         HTTP 404
  GET    /systems/not-a-uuid         HTTP 404
  GET    /controlstreams/not-a-uuid  HTTP 404
  ```
  Plan-05's §6 stop-clause ("If live returns 400 instead of 500, the
  maintainer has fixed it… stop") applies here in spirit: the 22P02
  channel never existed because the column type is text, not uuid.
- **Decision:** Do not file. No defect to report. Plan-05 is shelved;
  no report-05 produced.
- **Status:** **Closed — not a defect.**

### 9. Legacy `ToTimeRange` silent-discard year-0001 pattern
- **Source:** Issue #18 closure (adjacent finding).
- **Category:** Defect (P3 — narrow residual exposure).
- **Summary:** `internal/model/common_shared/time_range.go:159, 164, 173` still
  use `t, _ := time.Parse(...); startTime = &t` — error discarded AND `&t`
  taken regardless, producing a pointer to year-0001 instead of nil.
  Reachable from (a) `UnmarshalJSON`'s string-form branch at
  `time_range.go:119` (`*tr = ToTimeRange(s)`), so a JSON body of
  `"phenomenonTime":"junk/junk"` still silently corrupts; and
  (b) `history.go:39`. Two-part fix: inline-guard the three sites OR
  deprecate `ToTimeRange` in favor of `toTimeRangeStrict` and migrate the
  two callers.
- **Status:** Ready to file after closure pass.

### 10. `samplingFeature@id` retains silent-drop type assertion
- **Source:** Issue #19 closure (adjacent finding).
- **Category:** Defect (P3 — narrow).
- **Summary:** `internal/api/observation_handler.go` still decodes
  `samplingFeature@id` with `if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != ""`.
  A JSON number for that field is silently dropped. Narrower than the
  phenomenonTime/resultTime defect (no repository default-fill, so GET
  shows null rather than a plausibly-correct substitution), but same bug
  shape. Apply the `present && nil-check && type-assert-with-explicit-error`
  pattern that phenomenonTime/resultTime now use post-`1b2b614`.
- **Status:** Ready to file after closure pass.

### 11. `resultTime` / `phenomenonTime` empty-string conflates with missing
- **Source:** Issue #20 closure (T5 of original 7-row matrix).
- **Category:** Defect (P3 — UX residual).
- **Summary:** `resultTime: ""` passes `(string); ok` but skips the inner
  `if rtStr != ""` block, falls through to the late `IsZero()` guard, and
  produces `"resultTime is required"` rather than a distinct empty-string
  message. Same applies symmetrically to `phenomenonTime`. Trivial fix:
  add `if rtStr == "" { return nil, &decodeError{msg: "resultTime must be
  a non-empty ISO 8601 string"} }` between the `ok` check and the inner
  parse block. The post-`1b2b614` decoder reduced conflation from 6→1 to
  2→1; this closes the last gap.
- **Status:** Ready to file after closure pass.

### 12. `DatastreamDataComponent.Constraint` orphaned in validator
- **Source:** Issue #21 closure (adjacent finding, scoped out by eval).
- **Category:** Defect (P2 — spec conformance).
- **Summary:** `DatastreamDataComponent.Constraint` round-trips on GET but
  is not consulted by `validateDataComponentValue` in
  `internal/api/observation_schema_validation.go`. Clients declaring
  numeric constraints (min/max, allowed-values list) get no enforcement on
  observation POST. Same "model declares it, GET round-trips it, validator
  ignores it" shape that #21 closed for `nilValues`.
- **Status:** Ready to file after closure pass.

### 13. `DatastreamDataComponent.Updatable` orphaned in validator
- **Source:** Issue #21 closure (adjacent finding, scoped out by eval).
- **Category:** Defect (P3).
- **Summary:** `DatastreamDataComponent.Updatable` round-trips on GET but
  is not consulted by the validator on the PUT/PATCH path. Same shape as
  Constraint; cross-cutting check needed on the update handlers.
- **Status:** Ready to file after closure pass.

### 14. Approach 3a schema-side FK tags (optional hardening)
- **Source:** Issue #16 closure.
- **Category:** Enhancement.
- **Summary:** Add `foreignKey:` / `references:` /
  `constraint:OnDelete:RESTRICT` tags to natural-side relations on
  Observation, Command, SystemEvent, SystemHistoryRevision, Datastream's
  `system_id` projection, and `Deployment.parent_deployment_id`. Defense-in-
  depth on top of the Approach 3b application-layer checks already in place.
- **Status:** File only if maintainer wants the deeper hardening.

### 15. Inline `@link` absolutization for remaining 5 resource types
- **Source:** Issue #24 closure.
- **Category:** Enhancement (P3 — spec conformance, sibling of fix that landed).
- **Summary:** `d2d1347` removed `SystemLink` from the Datastream and
  ControlStream domain models and made the JSON formatter project
  `system@link` from `SystemID` via `ToFunctionalAssociationHref`, producing
  absolute URIs. The same wire-format projection pattern was not applied to
  the other 5 resource types with inline `@link` fields enumerated in the
  eval: System, Deployment, SamplingFeature, Observation, Command (17
  inline link properties total across 7 resource types per
  `internal/model/domains/`). Recommended: audit each of the 5 remaining
  resource types and apply the same domain-model-removal + formatter-projection
  treatment where applicable, or accept that some inline `@link` fields are
  user-supplied (round-trip-preserved) and only normalize on serialize.
- **Status:** Ready to file after closure pass.

### 16. Inline `@link` Type/Title/UID enrichment (residual from #25)
- **Source:** Issue #25 closure. Maintainer self-acknowledged: *"Mostly
  there for most associations however not all fully enriched"*.
- **Category:** Enhancement (P4 — UX, no spec violation).
- **Summary:** `3fa1b0c` and `704a9e3` populated server-generated `Rel`
  (`ogc-rel:*` vocabulary) on the supplementary `links[]` array.
  `d2d1347` reorganized inline `@link` as a wire-format projection
  carrying `Href` only. Three enrichment gaps remain on the inline
  `@link` emission across DS/CS and the other 5 resource types
  (System, Deployment, SamplingFeature, Observation, Command):
  (a) `Type` constants — cheap, no DB cost (e.g.
      `system@link → application/geo+json`,
      `procedure@link → application/sml+json`).
  (b) `Title` (= linked-resource Name) — requires read-time enrichment
      via batched preload in `SerializeAll` to avoid N+1.
  (c) `UID` (= linked-resource UID) — same enrichment path as Title.
  Sequence after item #15 (broader 5-resource-type absolutization
  audit) since both touch the same formatter sites.
- **Status:** Ready to file after closure pass.

### 17. ~~Strict JSON decoder rejects nested SensorML fields previously accepted; breaks OSHConnect-Python publishers~~ — **SUPERSEDED 2026-05-05**
- **Source:** Discovery finding 2026-05-05, surfaced during deployment-pinning of `cs-go-upstream` at upstream `df6da0d` and pilot of OSHConnect-Python publisher fleet against the new endpoint. No fork-side issue, no plan-NN.
- **Category:** ~~Defect (P2 — breaking wire-protocol regression).~~ **Re-categorized:** client-side bug in OSHConnect-Python publishers (latent silent SensorML field loss exposed by upstream `a467aba`'s strict decode). **Upstream `connected-systems-go` is correct; no upstream filing.**
- **Authoritative finding:** [`issue-evaluations/silent-sensorml-field-loss-pre-strict-decoder.md`](issue-evaluations/silent-sensorml-field-loss-pre-strict-decoder.md)
- **Disposition plan:** [`plan-report-13-disposition.md`](plan-report-13-disposition.md)
- **Original (parked) report:** [`upstream-issue-reports/_PARKED_report-13-strict-decoder-osh-publisher-bootstrap-regression.md`](upstream-issue-reports/_PARKED_report-13-strict-decoder-osh-publisher-bootstrap-regression.md) — framing inverted; preserved for forensic value only.
- **Original recommended fix (rejected):** sync `*GeoJSONProperties` wrapper structs with domain structs. Rejected because the GeoJSON-encoding path is spec-correctly stripped; widening it would conflate `application/geo+json` and `application/sml+json` against CSAPI Part 1.
- **Roundtrip evidence (2026-05-05):** POST with `Content-Type: application/json` carrying SensorML fields under `properties` → 201 on pre-strict server but `keywords` absent on GET (silent loss); same payload on `application/sml+json` with SensorML shape → 201 + GET preserves `keywords`. Strict server returns 400 on the broken shape (correctly).
- **Status:** **Superseded.** Action shifted to fork-side OSHConnect-Python repair per `plan-report-13-disposition.md`. Optional P4 upstream filing on content-type enforcement deferred.

---

## Closed / superseded

_(none yet)_

---

## Process notes

- Order of filing: file defects (1, 2, 3, 4, 5, 7) before enhancements (6, 8).
- Reference original evaluation paths under `docs/research/issue-evaluations/`
  when filing, so the maintainer can see the full context.
- Update this file as each item is filed: move from "Open items" → "Closed /
  superseded" with the new upstream issue number.
