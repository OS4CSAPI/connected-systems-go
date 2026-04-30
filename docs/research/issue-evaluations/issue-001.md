# Issue #1 — Datastream creation without explicit `uid` stores empty string, violates unique constraint on second create

- **URL:** <https://github.com/OS4CSAPI/connected-systems-go/issues/1>
- **State at evaluation:** open
- **Labels:** `bug`
- **Filed by:** Sam-Bolling
- **Filed:** 2026-04-17
- **Comments at evaluation:** 0
- **Evaluated against cs-go HEAD:** `1562201a44aa0dbd903ce44a1af40c7f662b2d4a` ("update datastreams", 2026-04-20)
- **Date:** 2026-04-30

---

## 1. Issue body — load-bearing claims

The issue makes the following factual claims that the verdict depends on:

| # | Claim | Source in issue |
|---|---|---|
| C1 | POSTing a datastream without a `uid` field stores `""` (empty string) in the database. | "Problem Statement" |
| C2 | The `uid` column has a unique constraint. | "Problem Statement" |
| C3 | A second create without `uid` fails with a PostgreSQL unique constraint violation referencing a constraint name. | "Problem Statement", "Actual behavior" |
| C4 | The `id` field is auto-generated server-side, providing a precedent for auto-generating `uid`. | "Option A — Recommended" |
| C5 | Workaround in production: publishers in `OS4CSAPI/OSHConnect-Python` always send explicit `uid` values. | "Workaround" |

Stated severity: P1-Critical. Stated category: API Design / Data Integrity. Stated ownership: connected-systems-go.

---

## 2. Verification

### 2.1 Code state at filing (commit `dacae7b`, 2026-04-17 and earlier)

Before the rebase performed during this evaluation, the working tree was at `c9747af`. Inspecting the diff `dacae7b..1562201` for `internal/model/domains/datastream.go`:

```diff
 type Datastream struct {
     Base
-    CommonSSN
+    Name        string `gorm:"type:varchar(255);not null" json:"name"`
+    Description string `gorm:"type:text" json:"description,omitempty"`
```

So at filing time, `Datastream` embedded `CommonSSN`, and `CommonSSN` (defined in [`internal/model/domains/common.go`](../../../internal/model/domains/common.go)) declares:

```go
type CommonSSN struct {
    UniqueIdentifier UniqueID `gorm:"type:varchar(255);uniqueIndex" json:"uid"`
    Name             string   `gorm:"type:varchar(255);not null" json:"name"`
    Description      string   `gorm:"type:text" json:"description,omitempty"`
}
```

Three observations against the issue's claims at this code state:

- **C1 (`""` stored on omitted `uid`):** Plausible-and-likely. `UniqueIdentifier` is a `UniqueID` (alias for `string`) with no `omitempty` and no `*` pointer. Go's zero value for string is `""`. JSON unmarshalling of a body that lacks the `uid` key will leave `UniqueIdentifier == ""`. GORM will then INSERT that empty string into the `unique_identifier` column.
- **C2 (unique constraint on `uid`):** Confirmed. `gorm:"...;uniqueIndex"` directive on `UniqueIdentifier`. GORM will emit a unique index on the `unique_identifier` column at AutoMigrate time. Verified by reading [`internal/model/domains/common.go`](../../../internal/model/domains/common.go) line 17.
- **C3 (second create fails on duplicate `""`):** Logical consequence of C1 + C2 — PostgreSQL unique indexes treat `''` as a value, not a NULL, so two inserts of `''` collide. I did not run a live reproduction (see §3 below), but this is a well-understood PostgreSQL semantics, not an implementation guess.

### 2.2 `id` precedent (C4)

[`internal/model/domains/common.go`](../../../internal/model/domains/common.go) lines 25–31:

```go
func (b *Base) BeforeCreate(tx *gorm.DB) error {
    if b.ID == "" {
        b.ID = uuid.New().String()
    }
    return nil
}
```

**C4 is accurate.** `Base.ID` is auto-generated to a UUID when omitted. `UniqueIdentifier` has no equivalent hook.

### 2.3 Code state at HEAD (commit `1562201`, 2026-04-20)

The "update datastreams" commit removed `CommonSSN` from `Datastream` and replaced it with explicit `Name` and `Description` fields. **The `UniqueIdentifier` field — and its `json:"uid"` tag, and its `uniqueIndex` directive — no longer exist on `Datastream` at HEAD.**

This was apparently a fix attempt for the issue (timing matches: filed Apr 17, commit Apr 20), but the issue was not closed and no commit message references it.

The fix is **incomplete in two visible ways:**

1. **Compile break.** [`internal/model/generators/generators_datastream.go`](../../../internal/model/generators/generators_datastream.go) lines 175–181 still construct `domains.Datastream` with a `CommonSSN: domains.CommonSSN{UniqueIdentifier: …}` literal. Against the current `Datastream` struct that no longer embeds `CommonSSN`, this is a Go compile error. The file has no build tag and is imported by `e2e/observations_test.go` line 15. I could not run `go build` (no Go toolchain on this evaluation machine — see §3) but the symbol mismatch is unambiguous from static reading.
2. **DB column drift.** GORM's `AutoMigrate` (called from [`internal/repository/repository.go`](../../../internal/repository/repository.go) line 52) **adds** columns and indexes but does not drop them. Any DB initialized against the prior `Datastream` schema retains the `unique_identifier` column and the unique index. Removing the Go field does not remove the column. Issue #12 ("Datastream `unique_identifier` enforces global uniqueness — should be scoped per parent system"), filed later, corroborates that the column is still present in real deployments.

So the surviving runtime situation depends on which DB state a server is running against:
- **Fresh DB initialized post-`1562201`:** No `unique_identifier` column would be added (struct no longer has the field). The bug C1/C2/C3 is gone — but at the cost of removing the field from the API surface entirely, which the issue did not propose.
- **DB initialized pre-`1562201`:** `unique_identifier` column persists with its uniqueIndex. INSERTs from current code do not populate it. Behavior depends on the column's NOT NULL / default constraints, which I cannot verify without DB introspection.

### 2.4 Spec reference

The issue cites *OGC 23-002 §9.2 — Datastream resource definition*. I do not have a verified copy of OGC 23-002 in the workspace and have not retrieved it for this evaluation. The issue's claim that "not all OGC spec examples include `uid`" is asserted, not proven, in the issue body. **Spec verification is deferred** — flagged as an unknown.

### 2.5 Related artifacts

- **Issue #12** in this same repo addresses the unique-identifier scope question and presupposes the column/constraint still exist, supporting the "DB column drift" reading above.
- **Workaround claim (C5)** — out of scope to verify in this evaluation; not load-bearing for the verdict.

---

## 3. Reproduction

**Not attempted.** Reasons:
- No Go toolchain on the evaluation machine, so building/running cs-go locally was not possible.
- Live test server `https://129-80-248-53.sslip.io/csapi-go` is the cs-go test instance but evaluating against it would require knowing which commit it is running and whether its DB was initialized pre- or post-`1562201`.

This is a known gap, recorded under §6 below.

---

## 4. Verdicts

| Dimension | Verdict | Rationale |
|---|---|---|
| **Validity** | `partially-confirmed` | Bug as described is **logically confirmed** at the commit current at filing (`dacae7b`-era) by static code reading: `UniqueIdentifier` with `uniqueIndex`, no auto-gen hook, no `omitempty`. Bug is **not directly reproducible against current HEAD** because the field has been removed from the struct, but the underlying DB column likely persists in real deployments and issue #12 confirms this. |
| **Legitimacy** | `defect` | Independent of OGC spec opinion: cs-go's own design intent — applying `uniqueIndex` to a field that has no auto-population path and no `omitempty`/pointer indirection — is internally inconsistent. The combination guarantees the second-insert collision the issue describes. |
| **Accuracy** | `accurate` (at filing time) / `stale` (now) | Every technical claim verifiable against `dacae7b`-era code held up. The issue body is now stale because a partial fix has landed without closing the issue or updating it. |
| **Completeness** | `complete` | Symptom, request shape, expected vs. actual behavior, three solution options with pros/cons, scope guardrails, acceptance criteria, dependencies, references — all present. The only completeness gap is that the issue could not have anticipated the partial fix in `1562201`. |

---

## 5. Reasoning summary

The issue is well-written and accurately described a real defect at the time it was filed. Three days later, commit `1562201` shipped what appears to be an *attempt* at fixing it — by removing `CommonSSN` from `Datastream` entirely — but:

1. The fix was not announced in the commit message ("update datastreams"), not linked to the issue, and the issue was not closed.
2. The fix introduced a compile-time inconsistency in the generators package.
3. The fix relies on GORM column-drop semantics it does not provide. Existing DB instances retain the constraint.
4. The fix removes the `uid` API surface entirely, which is more invasive than any of the three options the issue proposed (auto-generate / nullable / require). None of the three were chosen; a fourth path ("remove the field") was taken silently.
5. Issue #12, filed later, presupposes the column still exists and asks for its semantics to be changed — strong evidence the runtime situation has not actually been resolved end-to-end.

---

## 6. Recommendation

**Keep with edits.** The issue is legitimate and useful. Suggested edits to the issue body, in priority order:

1. **Add a status note at the top** stating that commit `1562201` ("update datastreams", 2026-04-20) removed `CommonSSN` from the `Datastream` struct and that this **was not announced as a fix for this issue** but appears to address part of it.
2. **Add a "Current state" section** describing the compile break in `generators_datastream.go` and the DB column drift risk.
3. **Cross-reference issue #12** as a sibling that tracks the "DB column / unique-index still exists" facet.
4. **Re-evaluate the proposed solutions** in light of what `1562201` actually did: is silent field removal the desired resolution, or should one of A/B/C be applied instead?

Do **not** close as fixed: the partial-fix state is worse than either (a) properly applying one of A/B/C or (b) properly reverting `1562201` and choosing a deliberate path.

---

## 7. Open questions / unknowns

1. **Spec authority.** Does OGC 23-002 §9.2 mandate a `uid`/`UniqueIdentifier` field on Datastream? The issue's three options assume cs-go has freedom; if the spec mandates the field, Option B (nullable) and the silent-removal path in `1562201` are both spec-violating.
2. **Live DB state.** Does the current cs-go test deployment at `129-80-248-53.sslip.io` have the `unique_identifier` column? If yes, the bug as described is still active there even though current `main` source code looks like it removed the field.
3. **Commit `1562201` intent.** Was it intended as a fix for this issue, or unrelated cleanup that happened to touch the same field? The commit message does not say. Asking the author would resolve this.
4. **Compile state of `main`.** Does `main` actually build today? Could not verify without a Go toolchain. If it doesn't build, the e2e tests aren't running, and CI signal on this repo is not what it appears.
5. **`?uid=` query parameter behavior (issue #7).** Issue #7 reports `?uid=` is silently ignored. If the server now has no `uid` field on Datastream at all, the connection between #1 and #7 may be tighter than either issue captures.
