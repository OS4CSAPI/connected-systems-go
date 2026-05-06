# Plan — Report-13 Disposition (Strict-decoder finding)

**Date:** 2026-05-05
**Status:** active
**Trigger:** roundtrip testing on 2026-05-05 inverted report-13's framing.
The strict-decoder 400 is not an upstream regression — it is correct
behavior surfacing pre-existing **silent SensorML field loss** in
OSHConnect-Python publisher bootstraps.

**Scope constraint (user directive 2026-05-05):** No work on
`Botts-Innovative-Research/OSHConnect-Python` upstream. All client-side
work happens on `OS4CSAPI/OSHConnect-Python` only. No PR back upstream.

---

## Evidence (verified 2026-05-05)

Roundtrip on `https://129-80-248-53.sslip.io/csapi-go/` (pre-strict
build at `c9747af`) and `/csapi-go-upstream/` (strict build at
`df6da0d`):

| # | Server | `Content-Type` | Payload shape | POST | Round-trip preserves `keywords`? |
|---|---|---|---|---|---|
| 1 | pre-strict (`c9747af`) | `application/json` | GeoJSON Feature with `keywords` under `properties` | **201** | **NO — silently dropped** |
| 2 | pre-strict (`c9747af`) | `application/sml+json` | SensorML (`type: PhysicalSystem`, top-level `uniqueId`, top-level `keywords`) | **201** | **YES** |
| 3 | strict (`df6da0d`) | `application/json` | GeoJSON Feature with `keywords` under `properties` | **400** (`unknown field 'keywords' in properties`) | n/a |

**Conclusion:** Upstream `a467aba` ("Adding Strict Parsing") is correct.
OSHConnect-Python publishers have been emitting wrong content-type +
wrong payload shape for procedures/deployments/systems since fleet
inception. All 10 publishers in the OS4CSAPI fleet have likely been
silently losing 9 SensorML metadata fields (`keywords`, `identifiers`,
`classifiers`, `characteristics`, `capabilities`, `contacts`,
`documentation`, `history`, `securityConstraints`, `legalConstraints`).

---

## Action plan

### A. Park `report-13` (today)

1. Rename `docs/research/upstream-issue-reports/report-13-strict-decoder-osh-publisher-bootstrap-regression.md`
   → `docs/research/upstream-issue-reports/_PARKED_report-13-strict-decoder-osh-publisher-bootstrap-regression.md`.
2. Prepend a banner: framing inverted, do not file upstream, see eval
   (step B) for actual finding.
3. Update `docs/research/upstream-followup-backlog.md` item **#17** —
   re-tag as **superseded** with cross-link to step B's eval.
4. Update `docs/research/report-audit-log.md` — append a `report-13`
   entry with **verdict: parked** and reasoning summary.
5. Commit + push.

### B. Internal evaluation (today)

6. Create `docs/research/issue-evaluations/silent-sensorml-field-loss-pre-strict-decoder.md`:
   - Roundtrip-evidence table (above).
   - SHA bisect: `a467aba` introduces strict decode; pre-strict
     persisted only `uid` / `name` / `description`.
   - File pointers: `procedure_geojson.go` decoder site;
     `ProcedureGeoJSONProperties` struct field set;
     `Procedure` domain struct field set.
   - 9-field list of silently-lost SensorML metadata.
   - Conclusion: upstream correct, OSHConnect-Python wrong.
7. Cross-link from the parked report banner.
8. Commit + push.

### C. Audit fork-side data integrity (today)

9. Query `cs-go` / `cs-go-head` Postgres for procedures whose persisted
   row has any of the 9 fields populated. Spot-check 5 publisher-bootstrapped
   rows; expect: all 9 fields NULL/empty.
10. Document findings in the eval (step B) under a "data-integrity audit"
    section. Confirms fleet prod data is recoverable only by re-bootstrap.
11. Update top-level todo: **"Wipe upstream DB and recreate"** is blocked
    by step E completion (publisher fix).

### D. File issue on `OS4CSAPI/OSHConnect-Python` (this week)

12. Open issue on `OS4CSAPI/OSHConnect-Python`:
    - **Title:** `[P1] Procedure/Deployment/System POSTs send GeoJSON Feature with SensorML metadata under properties; server silently drops 9 spec fields`
    - **Severity:** P1 — silent data loss in prod fleet, 10 publishers, since fleet inception.
    - **Body:** roundtrip table; client locations (`publishers/bootstrap_helpers.py:api_post`, `publishers/nws/nws/bootstrap_nws.py:122`, `:212`); fix shape (switch to `application/sml+json` + SensorML payload); cite upstream `a467aba` as surfacer.
    - **Validation chain:** link to internal eval (step B).
13. Do **not** open a PR back to `Botts-Innovative-Research/OSHConnect-Python`.

### E. Fix `OS4CSAPI/OSHConnect-Python` (this week)

14. Branch `fix/sml-content-type-and-shape` off `OS4CSAPI/OSHConnect-Python:main`.
15. Update `publishers/bootstrap_helpers.py`:
    - `api_post` accepts a `content_type` + `payload_shape` selector
      (default-back-compat for non-SensorML callers).
16. Refactor `publishers/nws/nws/bootstrap_nws.py`:
    - `ensure_procedure` / `ensure_deployment` / `ensure_system` build
      SensorML-shaped bodies (top-level `type: PhysicalSystem` /
      `Deployment` / `System`, top-level `uniqueId`, top-level
      `keywords` etc.).
    - POST with `Content-Type: application/sml+json`.
17. Apply same refactor to the 9 sibling publishers (one bootstrap
    file per publisher under `publishers/*/`).
18. Add `tests/test_bootstrap_roundtrip.py`:
    - POST procedure with `keywords` ∪ 8 other SensorML fields.
    - GET back; assert all 9 fields round-trip.
19. Run the smoke test against:
    - `cs-go-upstream` (`df6da0d`, strict) — must return 201, GET preserves all 9.
    - `cs-go` (`c9747af`, pre-strict) — must return 201, GET preserves all 9.
    - `cs-go-head` (`4b994212`) — must return 201, GET preserves all 9.
20. Merge to `OS4CSAPI/OSHConnect-Python:main`.
21. Tag a release on the fork (semver per the fork's existing tagging
    convention — defer to maintainer practice).

### F. Optional small upstream-CSAPI filing (this week, P4 — defer-or-skip)

22. *Optional.* File on `SomethingCreativeStudios/connected-systems-go`:
    - **Title:** `[P4] /procedures POST with Content-Type: application/json silently accepts SensorML-shaped payloads; recommend 415 or content-type assertion`
    - Argues: the strict decoder caught field-shape errors but not
      content-type/encoding errors; the same payload labeled
      `application/sml+json` would have round-tripped correctly.
    - Skip if 11 issues this week is enough.
23. **Decision:** defer. Re-evaluate after step E lands.

### G. Re-bootstrap fleet (next week)

24. Execute todo item **#1** ("Wipe upstream DB and recreate") with
    publisher fix in place.
25. Execute todo item **#2** ("Pilot bootstrap NWS against upstream")
    pointing the fixed NWS publisher at `cs-go-upstream`.
26. Smoke-verify via curl: GET a representative procedure on
    `cs-go-upstream`, confirm 9-field round-trip.
27. Execute todo items **#3–#5** (fan out, systemd units, fleet
    verification).

### H. Resume audit pipeline (after G)

28. Confirm no remaining queued upstream-issue-reports beyond the
    parked report-13 (queue ends at report-12 / upstream issue #11).
29. Future findings either go through the same `report-NN` template
    (if upstream-bound) or directly into `issue-evaluations/` (if
    fork/client-bound — like step B established as precedent).

---

## Critical path

`A → B → D → E → G`

- **Today:** A, B, C
- **This week:** D, E
- **Next week:** G
- **Skip:** F (optional)
- **Skip:** PR upstream to Botts-Innovative-Research (per directive)

## Out of scope

- `Botts-Innovative-Research/OSHConnect-Python` (any work).
- Any further upstream `connected-systems-go` filings beyond the optional F.
- The audit pipeline for reports beyond #13 (queue is empty).
- Schema migration on `connected-systems-go` (not needed; upstream is correct).
- Fork-side `connected-systems-go` patches (not needed).
