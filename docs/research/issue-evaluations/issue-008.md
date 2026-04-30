# Issue #8 — `/deployments` only returns top-level deployments

**Issue URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/8
**Type:** Reported as P2-Important
**Verdict:** **VALIDATED with corrections.** The spec-conformance gap is real and matches the canonical OGC OAS. P2 is appropriate (arguably P1 because of the additional `?id=` defect that breaks canonical URI lookup of subdeployments). Two of the issue's specific claims are factually wrong and need correction. The main fix proposed (Option A) is correct and additionally repairs two related defects the issue does not mention.

---

## 1. The core claim is correct

The default `/deployments` listing on cs-go **does** exclude subdeployments. Confirmed empirically:

```
T1: GET /deployments                  → numberMatched=1 (parent only, child invisible)
T4: GET /deployments/{parent}/subdeployments → numberMatched=1 (child)
T5: GET /deployments/{childId}        → 200 (child retrievable directly)
```

Code path (`internal/repository/deployment_repository.go` `applyFilters`, line ~198):

```go
} else { // parentId == nil (top-level endpoint)
    if params.Recursive {
        // empty
    } else {
        query = query.Where("parent_deployment_id IS NULL OR parent_deployment_id = ''")
    }
}
```

That `WHERE parent_deployment_id IS NULL` is the root cause.

## 2. The spec is unambiguous — and the asymmetry between `/systems` and `/deployments` is deliberate

Direct fetch of canonical Part 1 OAS at `opengeospatial/ogcapi-connected-systems`:

| | `paths/systems.yaml` | `paths/deployments.yaml` |
|---|---|---|
| GET description | "By default, **only top level systems** are included (i.e., subsystems are ommitted) unless the `parent` query parameter is set." | "List or search **all** `Deployment` resources available from this server endpoint." |
| `recursive` parameter | `$ref: ../parameters/recursive.yaml` (with override description) | **absent** |
| `parent` parameter | present | present |

And from `paths/subdeployments.yaml`:

> "individual members can also be retrieved by ID directly at the canonical `Deployment` resources endpoint."

This is a **direct spec guarantee** that subdeployments must be retrievable from `/deployments` by ID. cs-go violates this in two ways (T6 below).

The spec contract for `/deployments` GET, taken on its face, is:
- Default = flat listing of all deployments (no nesting filter).
- `?parent=X` = filter by parent ID list.
- Subdeployments retrievable by ID at the canonical endpoint.
- No `recursive` parameter at this path.

cs-go's default top-level-only filter is therefore a **non-spec restriction**, not a spec-aligned default.

## 3. The issue gets two things factually wrong

### 3a. "consistent with how `/systems` returns all systems (including subsystems)"

**False.** `internal/repository/system_repository.go:386-389`:

```go
func (r *SystemRepository) applyFilters(query *gorm.DB, params *queryparams.SystemsQueryParams) *gorm.DB {
    if !params.Recursive {
        query = query.Where("systems.parent_system_id IS NULL")
    }
```

`/systems` defaults to top-level only too. The difference is that `/systems`'s top-level-only default is **spec-mandated** ("By default, only top level systems are included"), while `/deployments`'s is **non-spec**. The framing "be consistent with /systems" is exactly backwards: the right framing is "the spec deliberately treats /deployments differently from /systems."

### 3b. "the `recursive=true` path for parentId==nil is a no-op"

**False.** Live test T2:

```
GET /deployments?recursive=true&limit=20
→ numberMatched=2 (parent + child)
```

The empty branch isn't a missing implementation; it's a (cryptically commented) implementation by omission. With `params.Recursive=true` and `parentId=nil`, no parent filter is applied at all — and since the default-branch's `parent_deployment_id IS NULL` lives in the sibling `else`, the query falls through unfiltered and returns everything. The only thing wrong here is the comment, which reads like a TODO.

(Note that `recursive` is not even a spec parameter at this endpoint, so this is a de-facto extension. It coincidentally produces the spec-correct flat listing if a client knows the magic word.)

## 4. Two additional defects under the same root cause

Both surfaced live; neither is mentioned in the issue.

### D1 — `?parent=X` returns 0 at top level (T3)

```
GET /deployments?parent=<parentId>
→ numberMatched=0
```

Static cause: `applyFilters` adds `WHERE parent_deployment_id IN (parentId)` for the `params.Parent` clause, then the same code path also adds `WHERE parent_deployment_id IS NULL`. AND'd, the condition is impossible. The spec-defined `parent` parameter is therefore broken at the canonical endpoint.

### D2 — `?id=<URI>` of a subdeployment returns 0 (T6)

```
GET /deployments?id=urn:test:issue8:dep:child
→ numberMatched=0
```

Static cause: the spec ID filter adds `WHERE id IN ? OR unique_identifier IN ?`; the top-level-only filter adds `WHERE parent_deployment_id IS NULL`. AND'd, the subdeployment is invisible to canonical URI lookup. **This directly violates the spec note in `paths/subdeployments.yaml`** ("individual members can also be retrieved by ID directly at the canonical `Deployment` resources endpoint"). For a publisher trying to reconcile a UID against the server (the `find_by_uid` workflow described in #7), this means subdeployments are simply unaddressable from the canonical endpoint by their identifier.

D2 is arguably more severe than the headline issue, because:
- The publisher pain in the original report (subdeployment idempotency check returning `None`) is a direct consequence of D2 specifically, not of the listing semantics.
- D2 violates an explicit spec sentence; the listing default violates only the spec's structural expectations (no `recursive` param + no top-level-only language).
- Pagination workarounds don't help — there is no value of `limit`/`offset` that surfaces a subdeployment to `?id=`.

Both D1 and D2 collapse into the **same single fix** as the issue's Option A: remove the unconditional `parent_deployment_id IS NULL` clause from the default top-level branch.

## 5. Recommended fix

**Option A from the issue body**, with the wording corrected to reflect spec authority:

```diff
 } else {
-    if params.Recursive {
-        // Canonical recursive search should include top-level deployments and all descendants.
-    } else {
-        query = query.Where("parent_deployment_id IS NULL OR parent_deployment_id = ''")
-    }
+    // Spec: GET /deployments returns ALL Deployment resources (paths/deployments.yaml
+    // does not include the `recursive` parameter or any top-level-only language,
+    // unlike paths/systems.yaml). Subdeployments must be retrievable from this
+    // canonical endpoint per paths/subdeployments.yaml. No parent filter applied here.
+    //
+    // params.Recursive is accepted as a no-op alias for backward compatibility,
+    // but is NOT a spec parameter at /deployments — clients should not rely on it.
+    _ = params.Recursive
 }
```

This single change:
- Aligns default listing with spec (closes #8 as filed).
- Fixes D1 — `?parent=X` works because the conflicting `IS NULL` clause is gone.
- Fixes D2 — `?id=<URI>` works for subdeployments at canonical endpoint (spec-mandated).

Option B in the issue body (preserve top-level-only default + implement `recursive=true`) is **not recommended**: (a) it perpetuates the non-spec default, (b) it does not fix D1 or D2 unless additional logic is added, and (c) it embeds a non-spec parameter as the only correct way to use the canonical endpoint.

## 6. Acceptance criteria (revised)

The original AC list is good; the additions are:

- [ ] **D1 (new):** `GET /deployments?parent={parentId}` returns the matching subdeployments (currently 0).
- [ ] **D2 (new):** `GET /deployments?id=<subdeployment-URI>` returns the matching subdeployment (currently 0).
- [ ] **Spec citation in code comment** — the new comment should reference `paths/deployments.yaml` and `paths/subdeployments.yaml` so the rationale is preserved against future "be consistent with /systems" refactors.
- [ ] Existing AC items unchanged (response includes `parent_deployment_id`; subdeployments endpoint still works; pagination correct; existing tests pass; E2E test added).

## 7. Severity

P2 as filed is reasonable. A case for P1 exists because D2 makes spec-compliant URI lookup of subdeployments fail at the canonical endpoint, which is a **direct violation of an explicit spec sentence** (not just a structural inference). I would not push for P1 unless a downstream consumer hits the D2 case specifically — the listing-default fix is what publishers need most, and any reasonable triage will land at the same one-line code change either way.

## 8. Methodology note

This is the first issue in the run that turned out to be **upstream-correct in its main thrust**, with the spec siding with the issue rather than against it. Two specific claims still failed scrutiny — the comparative ("/systems returns all systems") and the implementation diagnosis (`recursive=true` is a no-op) — but the underlying defect is real and well-evidenced.

Diligence rule that paid off here: **never accept a comparative-server claim without checking the canonical OAS for both endpoints by name.** In this case the canonical OAS is what made the determination clear: the spec author explicitly wrote different descriptions and different parameter sets for `/systems` and `/deployments`, and that asymmetry is the spec's actual position, not a copy-paste oversight.

Diligence rule that newly emerged: **when a code branch is documented as a no-op via comment, run it before believing the comment.** T2 took fifteen seconds and overturned a stated factual claim in the issue body.

## 9. Recommendation summary

- **Keep #8 open.** Its main thrust is correct and the fix is small and well-scoped.
- **Post comment** correcting the two factual errors, citing the spec asymmetry, and adding D1/D2 with live evidence so the fix is comprehensive.
- **Do not file a separate follow-up** for D1 or D2 — they share the same one-line fix and belong in the #8 PR.
- **Do file one follow-up** to track the issue's secondary discoverability claim if needed (deferred — the listing-default fix obsoletes it).
- This is the correct outcome for a defect that should be fixed upstream at `SomethingCreativeStudios/connected-systems-go`.
