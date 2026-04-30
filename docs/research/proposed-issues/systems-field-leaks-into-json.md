# Capital-`S` `Systems` GORM relationship field leaks into every datastream JSON response

> **Filing status:** Filed as [#15](https://github.com/OS4CSAPI/connected-systems-go/issues/15) on 2026-04-30. Drafted from issue #1 evaluation §2.8 T7.

| Field | Value |
|---|---|
| **Severity** | P3-Minor (response-shape pollution; no functional break) |
| **Category** | Code Quality / API Design |
| **Source** | Live empirical testing during issue #1 evaluation (parallel HEAD deploy) |
| **Ownership** | connected-systems-go (server) |
| **Affects** | cs-go HEAD `4b99421` (likely all commits since the field was added) |
| **Labels** | `code-quality`, `api-design`, `discovery-mode`, `good first issue` |

---

## Goal

The `Datastream.Systems` GORM `many2many` relationship field must not appear
in JSON responses. Add `json:"-"` to bring it in line with every other
relationship field in the codebase.

## Problem Statement

`Datastream` declares an explicit `Systems []System` field for the
`system_datastreams` join table, but unlike every comparable relationship
field in this codebase it is missing the `json:"-"` tag. Go's `encoding/json`
default behaviour is to emit unknown-tag fields under their Go name, so this
field shows up in every datastream response as `"Systems": null` (the slice
is not eagerly loaded by the read handlers).

**Affected code:**

```go
// internal/model/domains/datastream.go (line ~70, HEAD 4b99421)
type Datastream struct {
    Base
    // ... other fields, all properly tagged ...

    // Optional normalized IDs retained server-side for easier joins/filtering.
    SystemID            *string `gorm:"type:varchar(255);index" json:"-"`
    ProcedureID         *string `gorm:"type:varchar(255);index" json:"-"`
    DeploymentID        *string `gorm:"type:varchar(255);index" json:"-"`
    FeatureOfInterestID *string `gorm:"type:varchar(255);index" json:"-"`
    SamplingFeatureID   *string `gorm:"type:varchar(255);index" json:"-"`

    Systems []System `gorm:"many2many:system_datastreams;"`   // <-- missing json:"-"
}
```

**Codebase convention** (verified via `grep -RIn "json:\"-\"" internal/model/domains`):
the rest of the codebase already follows this pattern consistently — `system.go`,
`system_event.go`, `feature.go`, `control_stream.go` all tag their internal
relationship/foreign-key fields with `json:"-"`. The `Systems` field on
`Datastream` is the only deviation.

**Reproduction (from issue #1 §2.8 T7,
[`docs/research/evidence/issue-001/head-post-tests-2026-04-30.txt`](../evidence/issue-001/head-post-tests-2026-04-30.txt)):**

```bash
curl -s "$BASE/datastreams" \
  | jq '.items[0] | {has_Systems_capital: (has("Systems")), Systems_value: .Systems}'
# {
#   "has_Systems_capital": true,
#   "Systems_value": null
# }
```

**Impact:** Every datastream JSON response carries an extraneous capital-`S`
`Systems: null` property that is not part of the OGC API – Connected Systems
schema for `dataStream`. Strict consumers may reject it; lenient consumers
log a warning. The field is not user-facing data — it is a Go-side join helper.

## Suggested fix

Append `json:"-"` to the existing struct tag. One-line change.

```go
// from:
Systems []System `gorm:"many2many:system_datastreams;"`
// to:
Systems []System `gorm:"many2many:system_datastreams;" json:"-"`
```

## Files to modify

| File | Action | Rough size | Purpose |
|---|---|---|---|
| `internal/model/domains/datastream.go` | Modify | 1 line | Add `json:"-"` tag |
| `internal/api/datastream_handler_test.go` | Add | ~15 lines | Test that response JSON does not contain the `Systems` key |

## Scope — what NOT to touch

- ❌ Do **not** rename the field, change its Go visibility, or change its GORM
  tag.
- ❌ Do **not** add a lower-cased `systems` JSON property; the spec does not
  define one for `dataStream`.
- ❌ Do **not** sweep the rest of the codebase for similar leaks under this
  issue. If you find more, file separately, one issue per leaking type.
- ❌ Do **not** also add a database migration, repository method, or handler
  changes.

## Acceptance criteria

- [ ] `Systems` field in `internal/model/domains/datastream.go` has the
      `json:"-"` tag.
- [ ] `git grep -n 'Systems \[\]System' internal/model/domains/datastream.go`
      shows the line with `json:"-"`.
- [ ] Response from `GET /datastreams` does not contain a top-level `Systems`
      key on any item.
- [ ] New unit test asserts the absence of the `Systems` key in the
      marshalled JSON of a `Datastream` value.
- [ ] `go build ./cmd/server` exits 0.
- [ ] `go test ./internal/api/... ./internal/model/domains/...` exits 0.

### Acceptance gate (verification commands)

```powershell
# 1. Tag is present
git grep -n 'Systems \[\]System.*json:"-"' internal/model/domains/datastream.go
# expected: one match

# 2. No other Systems []System without json:"-" remains in the file
$bad = git grep -n 'Systems \[\]System' internal/model/domains/datastream.go `
       | Select-String -NotMatch 'json:"-"'
if ($bad) { throw "untagged Systems field still present: $bad" }

# 3. Build + tests
go build ./cmd/server
go test ./internal/api/... ./internal/model/domains/...
```

**Expected output:** the first command shows exactly one match, the second
produces no output, build/tests exit 0.

## Definition of done / closing workflow

1. Tick every acceptance-criteria checkbox.
2. Push the implementing commit to `OS4CSAPI/connected-systems-go` and copy
   its SHA.
3. Close with a summary comment containing: the commit SHA, files modified,
   acceptance-gate command output, and a `curl ... | jq 'has("Systems")'`
   sample showing `false` from a deployed instance.

## Dependencies

- **Blocked by:** none.
- **Blocks:** nothing.
- **Related:**
  - Issue #1 — surfaced this leak during live empirical testing.

## References

| # | Document | What it provides |
|---|---|---|
| 1 | [`internal/model/domains/datastream.go`](../../../internal/model/domains/datastream.go) line ~70 | The untagged `Systems` field |
| 2 | [`internal/model/domains/system.go`](../../../internal/model/domains/system.go), [`system_event.go`](../../../internal/model/domains/system_event.go), [`control_stream.go`](../../../internal/model/domains/control_stream.go), [`feature.go`](../../../internal/model/domains/feature.go) | Codebase convention: every other relationship/FK field uses `json:"-"` |
| 3 | [`docs/research/issue-evaluations/issue-001.md`](../issue-evaluations/issue-001.md) §2.8 T7 | Reproduction context |
| 4 | [`docs/research/evidence/issue-001/head-post-tests-2026-04-30.txt`](../evidence/issue-001/head-post-tests-2026-04-30.txt) | Raw `curl` evidence |
| 5 | OGC API – Connected Systems Part 2, bundled OAS31 — `dataStream` schema | Authority: no `Systems` property defined |

## Filing checklist

- [ ] Severity is P3 (no functional break, but ships with every response)
- [ ] Acceptance gate is fully automated and grep-/test-based
- [ ] Files-to-modify table is exact (one-line change + one new test)
- [ ] Definition of Done section is intact
- [ ] Dependencies block references #1
- [ ] References table includes the codebase-convention citation that makes
      this trivially uncontroversial
- [ ] Scope fence prevents creep into other types or migrations
- [ ] `good first issue` label is included (one-line, well-scoped fix)
- [ ] Title under ~120 characters and follows
      `<observed behaviour> <object>` form
