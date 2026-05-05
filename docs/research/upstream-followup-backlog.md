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
- **Category:** Defect (P2 — sibling of fix that landed on Datastream).
- **Summary:** `2dc09f7` added `json:"-"` to `Datastream.Systems` to prevent
  the GORM many2many slice from serializing into API responses. The exact
  same field exists on `ControlStream` and was not given the same tag, so
  `GET /controlstreams/{id}` still leaks the join data.
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

### 8. Malformed UUID path-param → 400 (currently 500)
- **Source:** Issue #17 closure.
- **Category:** Defect (P3 — UX / contract).
- **Summary:** `DELETE /datastreams/not-a-uuid` and similar produce SQLSTATE
  `22P02` (`invalid_text_representation`) at the DB, which falls through
  `errors.go`'s typed-sentinel switch (which only knows `ErrNotFound`,
  `ErrHasChildren`, `isFKViolation` for 23503) to the default-500 arm. Should
  return 400. `git grep -E 'uuid\.Parse|22P02|invalid_text_representation'
  upstream/main -- internal/api/` returns zero matches. Cleanest fix is
  `uuid.Parse(id)` at handler entry returning 400, short-circuiting the DB
  round trip. Affects all path-param UUID handlers (DELETE, GET-by-id, PUT,
  PATCH).
- **Status:** Ready to file after closure pass.

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

### 17. Strict JSON decoder rejects nested SensorML fields previously accepted; breaks OSHConnect-Python publishers
- **Source:** Discovery finding 2026-05-05, surfaced during deployment-pinning of `cs-go-upstream` at upstream `df6da0d` and pilot of OSHConnect-Python publisher fleet against the new endpoint. No fork-side issue, no plan-NN.
- **Category:** Defect (P2 — breaking wire-protocol regression).
- **Summary:** `a467aba` ("Adding Strict Parsing") switched the GeoJSON-wrapper deserializers (Procedure / Deployment / System) to `common_shared.DecodeWithFieldErrors`, which rejects unknown fields. Each `{Resource}GeoJSONProperties` wrapper struct in `internal/model/domains/{procedure,deployment,system}.go` is a strict subset of its corresponding domain struct, so spec-legitimate SensorML metadata fields (`keywords`, `identifiers`, `classifiers`, `characteristics`, `capabilities`, `contacts`, `documentation`, `history`, `securityConstraints`, `legalConstraints`, …) — historically accepted at the parent commit `c9747af` — now produce HTTP 400 `{"error":"unknown field 'X' in properties"}` deterministically. Documented OSHConnect-Python publishers (Botts-Innovative-Research/OSHConnect-Python and the OS4CSAPI fork; 10-publisher real-time fleet) cannot bootstrap procedures/deployments/systems against fresh-built `upstream/main`. Recommended fix: synchronise the three `*GeoJSONProperties` wrapper structs with their corresponding domain structs (declarative additions; no logic changes; preserves `a467aba`'s strict-mode guard).
- **Reported in:** [`upstream-issue-reports/report-13-strict-decoder-osh-publisher-bootstrap-regression.md`](upstream-issue-reports/report-13-strict-decoder-osh-publisher-bootstrap-regression.md).
- **Status:** Ready to file after closure pass.

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
