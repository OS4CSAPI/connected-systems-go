# Issue #7 — `?uid=` query parameter "silently ignored"

**Issue URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/7
**Type:** Reported as P1-Critical bug
**Verdict (after deep evaluation):** **Research-resolved. No P1 bug. The OGC CSAPI spec does not define a `uid` query parameter; the spec parameter is `id`, which already accepts both local IDs and URIs (UIDs), and cs-go implements it correctly. No code changes recommended against this issue. Optional low-priority enhancement (SensorHub-compat alias) noted below.**

---

## 1. The reported symptom is real

Verified live on HEAD `https://129-80-248-53.sslip.io/csapi-go-head/`:

```
GET /systems?uid=urn:test:issue1:sys:1
→ numberMatched: 12  (i.e. all systems on the server, paginated)
```

The `uid` query parameter is indeed not honored.

The static-analysis hits the issue describes are also correct:
- `internal/model/query_params/query_params.go` does not call `Get("uid")` (verified end-to-end across the whole `internal/` tree — zero hits for `Get("uid")` and zero for any `UIDs []string` field).
- Every per-resource `*QueryParams.BuildFromRequest` embeds the base via composition; none override.
- Every list-endpoint repository's `applyFilters` consumes `params.IDs` only.

So the *factual* description in the issue is accurate. The *interpretation* (that this is a P1 spec-compliance failure) is the part that does not survive scrutiny.

## 2. The spec does not define a `uid` query parameter

The issue cites: *"OGC 23-001 §7.3 defines `uid` as a standard query parameter for filtering resources by their globally unique identifier."*

Direct inspection of the **canonical OGC source** at `opengeospatial/ogcapi-connected-systems/master/api/part1/openapi/parameters/idList.yaml`:

```yaml
name: id
description: |-
  List of resource local IDs or unique IDs (URI).
  Only resources that have one of the provided identifiers are selected.
in: query
required: false
schema:
  $ref: idListSchema.yaml
explode: false
examples:
  id:
    summary: Local IDs
    value: 'RES_ID1,RES_ID2,RES_ID63'
  uri:
    summary: Unique IDs
    value: 'urn:example:resource:001,urn:example:resource:033'
```

The same shape is referenced from every list-endpoint YAML in Part 1 via `$ref: ../parameters/idList.yaml`, and is inlined identically in the Part 2 OAS bundle (`.tmp-csapi-part2.yaml` line 52-77 for `/datastreams`).

**The spec defines exactly one parameter — `id` — whose value list may contain either local IDs OR URIs (the things the issue calls "UIDs").** Grepping the Part 2 bundle for `name: uid` as a query parameter returns zero hits (the only `uid:` matches in the spec are property fields on Link / Feature objects, not query parameters).

There is no `?uid=` parameter in OGC 23-001 / 23-002.

## 3. cs-go implements the spec parameter correctly

Every list-endpoint repository's `applyFilters` does the spec-correct dual-match:

```go
// internal/repository/system_repository.go:393   (and equivalents in
// datastream / control_stream / deployment / procedure / property /
// sampling_feature / feature repositories)
if len(params.IDs) > 0 {
    query = query.Where("id IN ? OR unique_identifier IN ?",
                        params.IDs, params.IDs)
}
```

(`commands` / `observations` / `system_events` use only `id IN ?` because those resources don't carry their own UID per spec — verified against the domain models.)

This means **`?id=<URI>` is already the spec-compliant URI filter, and it works**.

## 4. Live confirmation

Full transcript: [`docs/research/evidence/issue-007/live-tests-head-2026-04-30.txt`](../evidence/issue-007/live-tests-head-2026-04-30.txt). Six-test summary:

| # | Request | numberMatched | Verdict |
|---|---------|---------------|---------|
| T1 | `?id=666ed3fe-...` (local UUID) | 1 | ✅ local-ID branch |
| T2 | `?id=urn:test:issue1:sys:1` (URI) | 1 | ✅ **URI branch — the publisher's actual need** |
| T3 | `?uid=urn:test:issue1:sys:1` | 12 | unknown-param — silently ignored (same as T7) |
| T4 | `?id=urn:does:not:exist` | 0 | ✅ empty collection on no match |
| T5 | `?id=666ed3fe-...,urn:does:not:exist` | 1 | ✅ mixed comma-separated |
| T7 | `?foo=bar&limit=2` | 12 | unknown-param `foo` ignored — identical mechanism to T3 |

The behavior in T3 is *not* "the `uid` parameter is broken" — it is "Go's `url.Values.Get` returns empty for any unknown key, just like `foo` in T7". `uid` is unknown from the spec's perspective; the spec parameter is `id`.

## 5. Verdicts on the issue's claims

| Claim | Verdict |
|-------|---------|
| "`QueryParams.BuildFromRequest()` parses `id` but never reads `uid`" | **Factually true** — but a non-defect, because `uid` is not in the spec |
| "OGC 23-001 §7.3 defines `uid` as a standard query parameter" | **Incorrect** — the spec parameter is `id` and it accepts both forms |
| "All collection list endpoints silently ignore `?uid=` and return unfiltered results" | **Factually true** — but consistent with how the API treats every unknown parameter, including misspellings |
| Severity P1-Critical | **Disagree** — no spec-compliance failure; the publisher's actual need (URI lookup) is fully supported via `?id=` |
| Acceptance criterion: "Comma-separated UIDs work" | **Already met** via `?id=urn:a,urn:b` (T5 verified) |
| Acceptance criterion: "Empty collection when no resource matches" | **Already met** via `?id=urn:does:not:exist` (T4 verified) |
| Recommended Option A: add `UIDs` field + per-repo `unique_identifier IN ?` branch | **Not recommended** — duplicates a code path the spec parameter already covers; introduces drift risk; touches every repository for no spec benefit |
| Recommended Option B: alias `uid` → `IDs` | **Possible as a P4 SensorHub-compat alias only** — single-line change; explicitly NOT a spec compliance fix |

## 6. The actual root cause of the publisher's pain

OSH SensorHub appears to accept `?uid=` (a SensorHub-era / SOS 2.0 idiom). The OSHConnect-Python publisher inherited this idiom and sent `?uid=urn:...` to cs-go, where it falls through silently. The publisher's correct fix:

```python
# bootstrap_helpers.py — corrected
def find_by_uid(session, url, target_uid):
    resp = session.get(f"{url}?id={target_uid}")   # spec parameter
    items = resp.json().get("items", resp.json().get("features", []))
    return items[0] if items else None
```

This is one character shorter than the workaround, drops the client-side filter loop, removes the `?limit=1000` correctness hazard, and works on cs-go today. The interop fix is on the **client** side, not the server.

## 7. The narrower defect that *is* present

Unknown query parameters are silently ignored across the API surface (verified by T7). For `?uid=` specifically this becomes a publisher-side discoverability footgun, but the same hazard exists for any misspelling: `?lmiit=10`, `?systemId=x`, etc. OGC API Common does not mandate strict-unknown-param rejection, so this is a **docs/UX** issue, not a bug.

## 8. Recommendation

**Close #7 as research-resolved.** The factual claims about cs-go's code are accurate; the spec interpretation is not. No `UIDs` field, no separate `uid` query parameter, no per-repo changes are warranted as spec-conformance work.

Optional small follow-ups (will file as separate enhancements at most P4):

- **F1 [P4 enhancement, optional]** — accept `uid` as a SensorHub-compat alias for `id` (Option B from the issue), single-line change in the base `BuildFromRequest`. Document explicitly that this is an interop convenience, not a spec parameter, so future maintainers don't propagate it as canonical. Decision pending — only worth filing if cross-stack interop with un-modifiable SensorHub clients is a project priority. Given the publisher fix is a one-character change (`uid` → `id`), the cleaner answer is "fix the client".
- **F2 [P4 docs]** — add a one-paragraph note in the OpenAPI doc / README that the spec filter parameter is `id` and accepts both local IDs and URIs, with examples. Removes the discoverability footgun for future publishers without committing to non-spec extensions. *Will file this one.*

## 9. Methodology note (preserve in changelog)

This was the third issue in a row (#5 → #6 → #7) where the comparative-server framing inverted the conformance polarity. SensorHub's behavior is being treated as the baseline; cs-go's deviation from SensorHub is being read as a defect. In all three cases, **direct spec inspection** showed that cs-go matches the spec and SensorHub deviates from it. The diligence rule that surfaced this:

> When an issue says "server X is missing feature Y that server Z has", do not stop at the comparative table. Open the canonical OGC source for the feature, find the parameter/field by name, and verify which server matches the spec. The asymmetry is then attributable to whichever side deviates.

For this issue specifically, fetching `opengeospatial/ogcapi-connected-systems/master/api/part1/openapi/parameters/idList.yaml` was the single decisive evidence — it shows `name: id` with the explicit "Local IDs OR Unique IDs (URI)" semantics, and there is no separate `uid` parameter anywhere in the spec.
