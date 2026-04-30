# Issue #9 — Default pagination limit of 10

**Issue URL:** https://github.com/OS4CSAPI/connected-systems-go/issues/9
**Type:** Reported as P3-Minor enhancement (label: `enhancement`)
**Verdict:** **Not a defect. cs-go matches the canonical CSAPI spec default exactly. The "unusually low" framing is contradicted by both the spec and the issue's own comparative table. Treat as a deployer-side UX preference: the upstream maintainer is free to raise the default toward the parent-spec value (100) as a UX choice, but doing so is a deviation, not a fix.**

---

## 1. Static claim is accurate

`internal/model/query_params/query_params.go`:

```go
params := &QueryParams{
    Limit:  10,
    Offset: 0,
}
```

Live confirmation:

| Test | Request | Result |
|---|---|---|
| baseline | `?limit=1000` | numberMatched=13, features=13 |
| T1 | default `GET /systems` | numberMatched=13, **features=10** |
| T2 | `?limit=5` | features=5 (override works) |

Default is 10. Override works. Pagination links present (assumed; not the disputed point).

## 2. Spec authority — cs-go matches the spec

Direct fetch of canonical CSAPI source `opengeospatial/ogcapi-connected-systems/master/api/part1/openapi/parameters/limit.yaml`:

```yaml
schema:
  type: integer
  minimum: 1
  maximum: 10000
  default: 10
```

The canonical CSAPI spec **explicitly sets default: 10**, and unlike the parent OGC API Features spec, the CSAPI limit.yaml does **not** include the "values are examples and can be changed" escape clause. cs-go's default of 10 is therefore literally spec-conformant.

For reference, the parent OGC API Features spec at `opengeospatial/ogcapi-features/master/core/openapi/parameters/limit.yaml`:

```yaml
schema:
  default: 100
description: |-
  ...
  Note that the values for minimum, maximum and default are examples and
  can be changed.
```

Two readings are defensible and both vindicate cs-go:
- **Strict:** CSAPI says default: 10 → cs-go matches → conformant.
- **Permissive:** the parent's escape clause makes any default permissible → cs-go's 10 is permitted; SensorHub's 100 is permitted → no defect either way.

There is no spec-compliance gap.

## 3. The issue's own comparative table refutes its own framing

From the issue body:

| Server | Default limit |
|---|---|
| SensorHub | 100 |
| pygeoapi | 10 |
| ldproxy | 10 |
| **Go CSAPI** | **10** |

cs-go matches **two of three** comparison servers and the canonical CSAPI spec. The "unusually low" claim concentrates entirely on SensorHub. This is the same SensorHub-as-baseline framing pattern that #5, #6, and #7 each exhibited — and in this case, the issue's *own data* contradicts the framing.

## 4. The compounding concern with #7 evaporated

Issue body: *"Once #7 is fixed, this becomes a lower-priority cosmetic issue."*

#7 was research-resolved (no fix needed; spec parameter is `id` and accepts URIs, so the publisher's `?limit=1000` workaround is unnecessary — the publisher should use `?id=urn:...` for one-character interop). With #7's underlying premise withdrawn, #9 sits at its stated cosmetic level alone.

The publisher's *actual* root cause (no pagination handling in `find_by_uid`) is a **client-side defect**, not a server-side one. Replacing `find_by_uid(GET /systems)` with `find_by_uid(GET /systems?id=urn:...)` is the same one-character fix flagged in #7 and removes any dependence on the default page size.

## 5. What about raising the default to 100?

The issue's recommended Option A (`Limit: 100`) deviates from the literal CSAPI spec but aligns with the OGC API Features parent-spec value. Whether to apply it is a **deployer/maintainer UX preference**, not a spec compliance call. Reasonable arguments on each side:

**For raising to 100:**
- Aligns with the OGC API Features parent default.
- Reduces foot-gun for naive clients that don't follow pagination.
- Matches SensorHub for cross-stack interop optics.

**Against raising to 100:**
- Deviates from the literal CSAPI spec default.
- pygeoapi and ldproxy also use 10 — cs-go is not an outlier in the OGC server ecosystem.
- The literal spec value 10 was almost certainly an intentional choice (the CSAPI authors removed the parent-spec's "examples can be changed" note, which signals deliberateness).
- Increases per-request memory/payload for default queries — modestly opinionated.

This is a judgment call best left to the upstream `SomethingCreativeStudios/connected-systems-go` maintainer.

## 6. Recommendation

**Treat as a P4-suggestion / wontfix candidate, not a defect.** Specifically:

- **Do not change the cs-go default unilaterally** — it is spec-conformant.
- **The publisher fix is to handle pagination correctly** (or use `?id=` for known-UID lookups per #7). Both fixes are client-side.
- **If the upstream maintainer wants to raise the default** to align with OGC API Features (100) for UX reasons, that is a reasonable deployer choice. We should not file this as a bug or a P3 against cs-go; we should at most raise it as a UX discussion item.

I will **not** file any follow-up issues against cs-go for this. There is nothing to fix. If we want to track the publisher-side pagination handling as work, that belongs in the OSHConnect-Python repo, not here.

## 7. Severity adjustment

Filed P3-Minor. Recommend **P4 / not-a-defect / close as wontfix-or-defer-to-maintainer-UX-call**. The issue's own self-downgrade ("cosmetic once #7 is fixed") plus #7's research-resolution puts this below the threshold for a meaningful action item.

## 8. Methodology notes

Three patterns recurring through this evaluation queue, all confirmed again by #9:

1. **SensorHub-as-baseline framing.** When SensorHub differs from the spec, that difference is being read as the spec. Three issues in a row for #5/#6/#7; #9 makes it four if we count cosmetic-severity ones. Continued diligence rule: open the canonical OGC source by parameter/field name before stopping at any comparative table — even (especially) the table the issue itself provides.
2. **Issue self-downgrades preempting our analysis.** #9's "becomes cosmetic once #7 is fixed" turned out to be exactly right after #7 was resolved. Worth checking issue cross-references early in any deep evaluation: dependencies that have already been resolved upstream or in our own queue can collapse the severity argument.
3. **Internal-evidence inconsistency.** When an issue's own table or example contradicts its own framing, that's a signal the framing is the load-bearing claim and the evidence is collateral. #9's table shows cs-go matches 2 of 3 comparison servers; the framing nonetheless says "unusually low". Always cross-check the evidence against the framing.
