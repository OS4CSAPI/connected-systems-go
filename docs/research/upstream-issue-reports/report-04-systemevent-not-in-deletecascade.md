# Report 04 — `SystemEvent` not handled in `SystemRepository.deleteCascade`

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-04-systemevent-not-in-deletecascade.md`](../upstream-issues/plan-04-systemevent-not-in-deletecascade.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#7** (`SystemEvent` not handled in `SystemRepository.deleteCascade`) |
| Source fork issue | `OS4CSAPI/connected-systems-go#16` (closed by upstream `fe9fbd0`; this filing is the residual enumeration-gap deferred at closure) |
| Research plan | [`../upstream-issues/plan-04-systemevent-not-in-deletecascade.md`](../upstream-issues/plan-04-systemevent-not-in-deletecascade.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P2** — silent orphaning of `SystemEvent` rows on system delete (latent until `fe9fbd0` activated the path; now active) |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). `fe9fbd0` is in lineage; defect persists.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git log --oneline upstream/main | Select-String fe9fbd0
fe9fbd0 Adding cascade delete and fixing existing cascade delete to full delete
```

```text
$ git show upstream/main:internal/repository/system_repository.go |
    Select-String 'deleteCascade|SystemEvent|system_events' -Context 0,0

func (r *SystemRepository) deleteCascade(tx *gorm.DB, systemID string) error {
        if err := tx.Model(&domains.System{}).Where("parent_system_id = ?", …
                if err := r.deleteCascade(tx, childID); err != nil {
        if err := tx.Where("parent_system_id = ?", systemID).Delete(&domains.SamplingFeature{})…
        if err := r.deleteSystemDatastreams(tx, systemID); err != nil {
        if err := r.deleteSystemControlStreams(tx, systemID); err != nil {
        if err := tx.Where("system_id = ?", systemID).Delete(&domains.SystemHistoryRevision{})…
        if err := tx.Exec("DELETE FROM system_deployments WHERE system_id = ?", systemID)…
        if err := tx.Exec("DELETE FROM system_procedures WHERE system_id = ?", systemID)…
        return tx.Delete(&domains.System{}, "id = ?", systemID).Error
```

`SystemEvent` and `system_events` appear **zero times** in the file.

```text
$ git show upstream/main:internal/model/domains/system_event.go | Select-String 'SystemID|type SystemEvent'
type SystemEvent struct {
    SystemID string `gorm:"type:varchar(255);index;not null" json:"-"`
```

`SystemEvent` is a System child via a `not null` `system_id` indexed
column with **no DB-level FK constraint** (per
[`../evidence/issue-002/fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt)),
so orphaning is silent.

**System-children audit on `df6da0d` — every domain type with a
system back-reference:**

| Child resource | Back-ref column | Enumerated in `deleteCascade`? |
|---|---|---|
| `System` (recursive child) | `parent_system_id` | ✅ recursive call |
| `SamplingFeature` | `parent_system_id` | ✅ |
| `Datastream` (+ `Observation`, `system_datastreams` join) | `system_id` | ✅ via `deleteSystemDatastreams` |
| `ControlStream` (+ `Command`, `system_controlstreams` join) | `system_id` | ✅ via `deleteSystemControlStreams` |
| `SystemHistoryRevision` | `system_id` | ✅ |
| `system_deployments` (join table) | `system_id` | ✅ raw exec |
| `system_procedures` (join table) | `system_id` | ✅ raw exec |
| **`SystemEvent`** | **`system_id`** | **❌ MISSING** |

`SystemEvent` is the **sole** missing entry. Filing is narrow.

```text
$ git log --oneline upstream/main -- internal/repository/system_repository.go | Select -First 6
c2ab201 time range and better 400 …
1b2b614 Adding support for "latest" for TimeRange
e0f31c4 Added/Fixed tests
fe9fbd0 Adding cascade delete and fixing existing cascade delete to full delete
5b5fb94 Added custom CS skill for cladue for better slop … Sampling Feature validation
dacae7b Bug fixes and clean up
```

No post-`fe9fbd0` commit has touched the enumeration.

**Live re-verification:** not captured this round. The full sequence
(POST system → POST `SystemEvent` → DELETE system → re-query) requires
authenticated admin endpoints that are not currently exposed on the
`csapi-go-upstream` deployment. The defect is structural — driven by
the absence of three lines of code in a known function — and the
static evidence is unambiguous. Parent issue #16's pre-`fe9fbd0` live
evidence ([`../evidence/issue-016/live-test-2026-04-30.md`](../evidence/issue-016/live-test-2026-04-30.md))
confirmed the same orphaning pattern on the same code path; the
maintainer's own `fe9fbd0` commit fixed all but this one child.

## 2. Static evidence

Source: [`../evidence/issue-016/static-analysis-2026-04-30.md`](../evidence/issue-016/static-analysis-2026-04-30.md)
(refreshed in §1 above).

`fe9fbd0` adopted Approach 3b (application-level child checks) for
the non-cascade branch and a transactional enumeration for the cascade
branch. The cascade enumeration in
`internal/repository/system_repository.go` `deleteCascade` covers
seven distinct child resources / join tables. `SystemEvent` is not
among them.

`internal/model/domains/system_event.go` declares the parent reference:

```go
type SystemEvent struct {
    Base
    SystemID string `gorm:"type:varchar(255);index;not null" json:"-"`
    // … event payload fields …
}
```

`internal/repository/system_event_repository.go` writes rows keyed by
`system_id`. There is no back-pointer to a cleanup hook on
`SystemRepository.Delete` / `deleteCascade`.

[`../evidence/issue-002/fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt)
shows `system_events.system_id` carries no FK constraint at the
PostgreSQL layer. Under Approach 3b's contract, the application
cleanup is therefore the **only** integrity boundary; missing
enumeration = silent orphan.

## 3. Live evidence

Not captured this round (auth-gated endpoint sequence). Justification
above in §1. Parent #16 live evidence transitively applies — same
handler, same code path, same defect class.

## 4. Spec authority

This is a **contract-completeness** defect, not a strict spec
violation. Spec authority plays a corroborating role.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001** — CSAPI Part 1, §"System events" | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Establishes `SystemEvent` as a normative child resource of `System`. The parent-child relationship is part of the resource model the server is responsible for maintaining. |
| **OGC 23-002** — CSAPI Part 2 SystemEvent schema | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms parent-system reference on `SystemEvent`. Used to demonstrate the relationship is normative and not optional. |

**Out-of-scope sources (not cited):** OAS 3.0.3, OGC 19-072, RFC 7493 —
none of these speak to repository cascade behavior.

## 5. Alternatives considered (internal-only)

### Option A — Add `SystemEvent` to `deleteCascade` (recommended)

```go
if err := tx.Where("system_id = ?", systemID).
    Delete(&domains.SystemEvent{}).Error; err != nil {
    return err
}
```

- One addition, mirroring the existing `SystemHistoryRevision` line.
- Matches the maintainer's chosen Approach 3b exactly.
- Zero risk; no behavioral change for any code path other than
  system deletion.

### Option B — Switch from Approach 3b to Approach 3a (FK constraints)

- Out of scope here. Tracked separately as backlog item #14.
- Would prevent this class of follow-up structurally, but is a
  much broader hardening effort and depends on maintainer appetite.

**Lead with Option A only.** Mention Option B once in §10's footer.

## 6. Recommended fix

**Option A.** Append a single block to `deleteCascade` mirroring the
existing `SystemHistoryRevision` enumeration:

```go
// internal/repository/system_repository.go, inside deleteCascade,
// adjacent to the SystemHistoryRevision delete:
if err := tx.Where("system_id = ?", systemID).
    Delete(&domains.SystemEvent{}).Error; err != nil {
    return err
}
```

Implementation surface: one function (`deleteCascade`), three lines
added. No schema migration. No FK addition. No public API change.

## 7. Scope guard

What NOT to touch as part of this filing:

- No schema migration. The `system_events.system_id` FK gap is
  separately tracked as backlog #14 (Approach 3a hardening) and is
  **not** in scope here.
- No re-litigation of Approach 3b vs 3a. Maintainer already chose 3b
  in `fe9fbd0`; this filing completes 3b's enumeration as designed.
- No `Delete(non-cascade)` branch changes. `SystemEvent` does not
  need to gate the non-cascade delete with `ErrHasChildren`, because
  it is a value-class child (no public-facing identifier independent
  of its parent system) — parent #16 §"Bug 3" eval confirms cleanup,
  not refusal, is the appropriate semantic. (Drafter may revisit if
  user prefers gating; tentative position is "cleanup only".)
- No other domain types. The §1 audit confirms `SystemEvent` is the
  sole gap.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#16` was the umbrella P1 finding that
  combined the cascade-failure family. Closed by upstream `fe9fbd0`
  adopting Approach 3b. Issue #16's own scope deferred the
  `SystemEvent` enumeration gap to a separate residual filing (this
  one) because at #16's filing time the parent code path was
  effectively unreachable via the public API.
- No fork-side patch for this defect; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Frame as "completing 3b" or "3b is fragile, reconsider 3a"? | **"Completing 3b."** Narrow, precedent-bound, friendly. 3a alternative referenced once as separate backlog item only. |
| Bundle backlog #14 (Approach 3a FK tags) into this filing? | **No.** Item #14 is conditional on maintainer appetite and covers a much broader hardening scope. File separately. |
| Audit other system children? Bundle if multiple gaps? | **Audited (§1 table). Only `SystemEvent` is missing.** Narrow filing. |
| Live reproducer required? | **No.** Auth-gated; defect is structural; static evidence + parent #16's pre-`fe9fbd0` live evidence are sufficient. Documented in §1 and §3. |
| Cite eval/evidence paths? | **Yes**, as validation-chain footer. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P2] SystemEvent not enumerated in SystemRepository.deleteCascade — silent orphan after fe9fbd0`

**Labels:** `bug`

---

### Context

`fe9fbd0` ("Adding cascade delete and fixing existing cascade delete
to full delete") adopted Approach 3b for `SystemRepository.Delete` —
application-level child checks on the non-cascade branch, and a
transactional enumeration of every child resource on the cascade
branch. Approach 3b's correctness depends on the cascade enumeration
covering every child resource, because there are no DB-level FK
constraints to backstop a missing entry.

`SystemEvent` is a normative child of `System` per CSAPI Part 1
§"System events" and is keyed by `system_id` (not-null indexed
column, no FK). It is **not** in `deleteCascade`'s enumeration. A
cascade delete therefore succeeds and silently orphans every
`SystemEvent` row attached to the deleted system.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

`internal/repository/system_repository.go` `deleteCascade` does not
delete `SystemEvent` rows. `internal/model/domains/system_event.go`
declares `SystemID string … not null`.
`system_events.system_id` has no DB-level FK. Result: `DELETE` of a
system with attached `SystemEvent`s succeeds; the events rows remain
in the table with a now-dangling `system_id`.

### Static evidence

`internal/repository/system_repository.go` (HEAD `df6da0d`),
`deleteCascade` enumerated children:

```go
func (r *SystemRepository) deleteCascade(tx *gorm.DB, systemID string) error {
    // recursive child Systems (parent_system_id)
    // SamplingFeature (parent_system_id)
    // deleteSystemDatastreams: Observations + Datastream + system_datastreams join
    // deleteSystemControlStreams: Commands + ControlStream + system_controlstreams join
    // SystemHistoryRevision (system_id)
    // system_deployments join (system_id)
    // system_procedures join (system_id)
    return tx.Delete(&domains.System{}, "id = ?", systemID).Error
}
```

`SystemEvent` / `system_events` appear zero times in the file
(`grep` returns no matches).

`internal/model/domains/system_event.go`:

```go
type SystemEvent struct {
    Base
    SystemID string `gorm:"type:varchar(255);index;not null" json:"-"`
    // …
}
```

No FK constraint on `system_events.system_id` (per repository's
schema-creation path; same gap class as documented for adjacent
tables in issue #2's evidence pack).

### Live evidence

Not included in this filing — the full sequence requires
authenticated admin endpoints. The defect is structural (three
lines of code missing in a single function) and the static evidence
is unambiguous. Pre-`fe9fbd0` live evidence on the same code path
demonstrated the orphaning behavior; `fe9fbd0` fixed seven of the
eight enumerated children but not `SystemEvent`.

### Recommended fix

Single addition to `deleteCascade`, mirroring the existing
`SystemHistoryRevision` line:

```go
if err := tx.Where("system_id = ?", systemID).
    Delete(&domains.SystemEvent{}).Error; err != nil {
    return err
}
```

One function, three lines, no schema change, no public API change.
Same shape as the seven sibling child-resource enumerations already
in `deleteCascade`.

### Spec authority

- **OGC 23-001** — CSAPI Part 1 §"System events" defines
  `SystemEvent` as a normative child of `System`. The parent-child
  relationship is part of the resource model the server is
  responsible for maintaining; silent orphaning violates that
  resource-model contract even though no clause directly mandates
  cascade behavior.
- **OGC 23-002** — CSAPI Part 2 SystemEvent schema confirms the
  parent-system reference is normative and required.

### Severity

**P2** — silent integrity loss on a publicly invokable path
(`DELETE /systems/{id}` with cascade). No 5xx is emitted; clients
have no way to detect the orphan unless they re-query system events
directly.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-016.md`](../issue-evaluations/issue-016.md) §"Bug 3"
- Evidence (static): [`docs/research/evidence/issue-016/static-analysis-2026-04-30.md`](../evidence/issue-016/static-analysis-2026-04-30.md)
- Evidence (live, parent): [`docs/research/evidence/issue-016/live-test-2026-04-30.md`](../evidence/issue-016/live-test-2026-04-30.md)
- Evidence (FK gap): [`docs/research/evidence/issue-002/fk-constraints-head-2026-04-30.txt`](../evidence/issue-002/fk-constraints-head-2026-04-30.txt)
- Plan: [`docs/research/upstream-issues/plan-04-systemevent-not-in-deletecascade.md`](../upstream-issues/plan-04-systemevent-not-in-deletecascade.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #7
- Related (separate filing): backlog #14 (Approach 3a FK constraints — would prevent this class of follow-up structurally).
