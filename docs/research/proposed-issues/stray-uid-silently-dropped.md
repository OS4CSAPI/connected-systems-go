# `POST /systems/{id}/datastreams` silently drops stray top-level `uid` field

> **Filing status:** Filed as [#13](https://github.com/OS4CSAPI/connected-systems-go/issues/13) on 2026-04-30. Drafted from issue #1 evaluation §2.8 T6.

| Field | Value |
|---|---|
| **Severity** | P2-Important |
| **Category** | API Design / Data Integrity |
| **Source** | Live empirical testing during issue #1 evaluation (parallel HEAD deploy) |
| **Ownership** | connected-systems-go (server) |
| **Affects** | cs-go HEAD `4b99421` (and every commit since `1562201`) |
| **Labels** | `bug`, `api-design`, `discovery-mode` |

---

## Goal

`POST /systems/{id}/datastreams` must reject (or at minimum warn on) request
bodies that include a top-level `uid` field that the server cannot persist,
instead of silently discarding it and returning `201 Created`.

## Problem Statement

Commit
[`1562201`](https://github.com/OS4CSAPI/connected-systems-go/commit/1562201a44aa0dbd903ce44a1af40c7f662b2d4a)
removed the `CommonSSN` embedded struct from `Datastream`, eliminating the
`uid` field from both the struct and the database table (verified live;
see issue #1 §2.6, §2.8). The current struct
([`internal/model/domains/datastream.go`](../../../internal/model/domains/datastream.go)
lines 14–73) has no `uid` field and no `json:"uid"` tag. Go's `encoding/json`
decoder ignores unknown JSON keys by default, so a request body containing
`"uid": "<anything>"` is parsed successfully and the `uid` value is silently
discarded.

This is a **silent data-loss surface** for any client written against the
pre-`1562201` API (production at
`https://129-80-248-53.sslip.io/csapi-go/` is still such a client, per issue
#1 §2.7): the client sends a `uid`, the server returns `201 Created`, the
client believes its `uid` was accepted, and on subsequent reads the field is
gone.

**Affected code:**

```go
// internal/model/domains/datastream.go (lines 14-73, HEAD 4b99421)
type Datastream struct {
    Base
    Name        string `gorm:"type:varchar(255);not null" json:"name"`
    Description string `gorm:"type:text" json:"description,omitempty"`
    // ... no uid field anywhere in the struct ...
}
```

```go
// internal/api/datastream_handler.go CreateDatastream
// uses h.fc.Deserialize, which delegates to encoding/json (or geojson formatter)
// for the Datastream type. encoding/json.Decoder.Decode does not error on
// unknown fields unless DisallowUnknownFields() is called.
```

**Reproduction (from issue #1 §2.8 T6, transcript at
[`docs/research/evidence/issue-001/head-post-tests-2026-04-30.txt`](../evidence/issue-001/head-post-tests-2026-04-30.txt)):**

```bash
curl -isS -X POST "$BASE/systems/$SYS_ID/datastreams" \
    -H "Content-Type: application/json" \
    -d '{
      "name":"DS C with stray uid",
      "uid":"urn:test:issue1:ds:stray",
      "observedProperties":[{"definition":"http://example.org/prop/pressure","label":"Pressure"}],
      "phenomenonTime":{"begin":"2026-04-30T00:00:00Z","end":null},
      "resultTime":{"begin":"2026-04-30T00:00:00Z","end":null},
      "resultType":"Measure",
      "live":false,
      "outputName":"pressure"
    }'
# HTTP/2 201
# location: .../datastreams/186b7853-786a-44d3-9489-bfe5c4f891e9

curl -s "$BASE/datastreams" | jq '.items[] | select(.id=="186b7853-786a-44d3-9489-bfe5c4f891e9")'
# Result: object has no "uid" key. The submitted "uid" is gone.
```

**Impact:** Any client built or tested against pre-`1562201` cs-go (which
includes the production deployment at the time of writing) will, after a
deploy of HEAD code without coordination, send `uid` fields that disappear
without notice. There is no warning header, no `Sunset`/`Deprecation` header,
and no error response. Issue #1 §2.3 establishes that the OGC 23-002 Part 2
spec does not mandate `uid` on `dataStream`, so the field's removal is
defensible — but the silent-acceptance behaviour is not.

## Suggested fix (one of)

The fix authority sits with the cs-go maintainer; the discovery is what
matters here. For reference, possible directions are:

1. **Reject unknown fields:** flip `json.Decoder.DisallowUnknownFields()` on
   the Datastream deserializer. Returns `400 Bad Request` with a structured
   error pointing at the offending key. Maximally explicit, but a breaking
   change for any client sending fields cs-go silently accepted before.
2. **Reject only `uid` specifically:** add an explicit pre-decode check for a
   top-level `"uid"` key and respond `400 Bad Request` with a message that
   names the issue and links to the spec. Targeted; preserves silent-accept
   for other unknown fields if that's intentional.
3. **Re-add a no-op `uid` field** with `json:"uid,omitempty"` and `gorm:"-"` so
   the value round-trips through the response (without persistence) until a
   later release decides between (1) and (2). Lowest blast radius; matches the
   current production wire shape.

## Files to modify

| File | Action | Rough size | Purpose |
|---|---|---|---|
| `internal/api/datastream_handler.go` | Modify (`CreateDatastream`, `UpdateDatastream`) | ~10–20 lines | Apply chosen rejection / pass-through logic |
| `internal/model/domains/datastream.go` | Modify if option 3 chosen | ~3 lines | Add no-op `uid` field |
| `internal/api/datastream_handler_test.go` | Add | ~30–60 lines | Test for the new behaviour on POST and PUT |

## Scope — what NOT to touch

- ❌ Do **not** re-introduce a database `uid` column or `unique_identifier`
  unique index. Issue #1 §2.6 establishes that the column is dropped from
  fresh installs intentionally.
- ❌ Do **not** alter the `Datastream` struct's other JSON tags or field order.
- ❌ Do **not** also fix issue #1's broader trajectory (test-fixtures break,
  DB migration plan, deployment sync). Those have separate trackers.
- ❌ Do **not** silently drop unknown fields on **other** resource types as
  part of this issue — scope is `Datastream` create / update only.

## Acceptance criteria

- [ ] POST or PUT to `/systems/{id}/datastreams` with a top-level `uid` in
      the body produces the chosen behaviour (400, 201-with-warning, or
      201-with-uid-round-trip — whichever option the maintainer selects).
- [ ] Existing GET responses are unchanged for records that were created
      without a `uid`.
- [ ] New unit test in `internal/api/datastream_handler_test.go` exercises
      the new behaviour.
- [ ] `go build ./cmd/server` exits 0.
- [ ] `go test ./internal/api/...` exits 0.
- [ ] `go vet ./...` exits 0.

### Acceptance gate (verification commands)

```powershell
go build ./cmd/server                   # must exit 0
go test ./internal/api/...              # must exit 0; new test must be green
go vet ./internal/api/... ./internal/model/domains/...  # must exit 0
```

**Expected output:** all three commands exit 0; new test name appears in the
`go test -v ./internal/api/...` listing.

## Definition of done / closing workflow

> An issue is not complete until it is closed with a summary comment.

1. Tick every checkbox above (`[ ]` → `[x]`).
2. Push the implementing commit to `OS4CSAPI/connected-systems-go` and copy
   its SHA.
3. Close with a summary comment containing: the commit SHA, the list of
   files modified (matching the table above), the acceptance-gate command
   output, and any deviation from the suggested fix options (or "no
   deviation").

## Dependencies

- **Blocked by:** none.
- **Blocks:** nothing currently filed; loosely informs any future migration
  plan emerging from issue #1.
- **Related:**
  - Issue #1 — Datastream `uid` empty-string collision. This issue documents
    one of the smaller defects introduced by issue #1's silent partial fix
    (`1562201`).
  - Issue #12 — `unique_identifier` global-uniqueness scope. Same family of
    `uid`-on-Datastream concerns.

## References

| # | Document | What it provides |
|---|---|---|
| 1 | [`internal/model/domains/datastream.go`](../../../internal/model/domains/datastream.go) lines 14–73 | Current `Datastream` struct (no `uid` field) |
| 2 | [`internal/api/datastream_handler.go`](../../../internal/api/datastream_handler.go) `CreateDatastream`, `UpdateDatastream` | Affected handler functions |
| 3 | [`docs/research/issue-evaluations/issue-001.md`](../issue-evaluations/issue-001.md) §2.8 T6 | Reproduction context |
| 4 | [`docs/research/evidence/issue-001/head-post-tests-2026-04-30.txt`](../evidence/issue-001/head-post-tests-2026-04-30.txt) | Raw `curl -isS` transcript |
| 5 | OGC API – Connected Systems Part 2, bundled OAS31 — `dataStream` schema | Spec authority: no `uid` property mandated |
| 6 | Commit [`1562201`](https://github.com/OS4CSAPI/connected-systems-go/commit/1562201a44aa0dbd903ce44a1af40c7f662b2d4a) | Origin of the silent-drop behaviour |

## Filing checklist

- [ ] Severity reflects "silent data loss" weight (P2 chosen)
- [ ] Acceptance gate is 100% automated (no manual-review steps)
- [ ] Files-to-modify table matches the suggested-fix options
- [ ] Definition of Done section is intact
- [ ] Dependencies block names #1 and #12
- [ ] References table includes spec citation, evidence transcript, and
      affected code paths
- [ ] Scope fence forbids re-introducing the column / unique index
- [ ] Title format: `<verb> <object> <observed behaviour>` and stays under
      ~120 characters
