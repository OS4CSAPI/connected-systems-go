# Report 06 — Legacy `ToTimeRange` silent-discard year-0001 pattern

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-06-totimerange-year-0001-silent-discard.md`](../upstream-issues/plan-06-totimerange-year-0001-silent-discard.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#9** (legacy `ToTimeRange` silent-discard year-0001 pattern) |
| Source fork issue | `OS4CSAPI/connected-systems-go#18` (closed by upstream `c2ab201`; this filing is the "adjacent finding" deferred at closure) |
| Research plan | [`../upstream-issues/plan-06-totimerange-year-0001-silent-discard.md`](../upstream-issues/plan-06-totimerange-year-0001-silent-discard.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P3** — narrow residual exposure (parent fix already covers high-traffic array/object decode paths; remaining channel is the string-form path + `history.go`) |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Parent fix is `c2ab201` ("time range and
better 400 …"). Defect persists in `ToTimeRange` itself.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git log --oneline upstream/main -- internal/model/common_shared/time_range.go | Select -First 4
c2ab201 time range and better 400 (now includes more info and field info)
1b2b614 Adding support for "latest" for TimeRange
554ada5 Moving fully to go_geom TimeRange now should work more OGC Like (not for sensorML though)
6b856bb Adding maybe better structured code
```

Legacy `t, _ := time.Parse(...)` sites in
`internal/model/common_shared/time_range.go` (line numbers on `df6da0d`):

```text
 54: func (tr *TimeRange) UnmarshalJSON(b []byte) error {
119:         *tr = ToTimeRange(s)             <-- delegation site (string-form)
129: func ToTimeRange(timeValue string) TimeRange {
147:         t, _ := time.Parse(time.RFC3339, parts[0])   <-- legacy #1 (start, 2-part)
152:         t, _ := time.Parse(time.RFC3339, parts[1])   <-- legacy #2 (end,   2-part)
161:         t, _ := time.Parse(time.RFC3339, parts[0])   <-- legacy #3 (single)
```

Note: backlog cited lines 159, 164, 173. Drift to 147/152/161 is from
`c2ab201`'s edits to neighbouring lines; the pattern itself is
unchanged. `c2ab201` actually **added** legacy site #3 (the
"single value" branch) at the same time it strict-parsed
`UnmarshalJSON` array/object branches.

`UnmarshalJSON`'s array branch (line 73-89) and object branch (line
98-115) both return `fmt.Errorf("invalid time …: must be RFC3339 …")`
on parse failure — parent fix verified in place.

```text
$ git grep -nE 'ToTimeRange\(' upstream/main -- '*.go'
internal/model/common_shared/history.go:39:         tr := ToTimeRange(s)
internal/model/common_shared/time_range.go:119:    *tr = ToTimeRange(s)         # UnmarshalJSON string-form
internal/model/common_shared/time_range.go:298:    return ToTimeRange(v)        # ParseTimeRange(string)
internal/model/common_shared/time_range.go:314:    return ToTimeRange(startStr + "/" + endStr)   # ParseTimeRange(map)
internal/model/common_shared/time_range_test.go:112:   tr := ToTimeRange("latest")
```

**Caller-graph correction.** Backlog said two callers
(UnmarshalJSON string-form + `history.go:39`). The actual reachable
set on `df6da0d` is **four** non-test callers:

| Site | Path |
|---|---|
| `time_range.go:119` | `UnmarshalJSON` string-form branch (`*tr = ToTimeRange(s)`) — JSON request bodies of `"phenomenonTime":"junk/junk"` |
| `time_range.go:298` | `ParseTimeRange(string)` adapter (`return ToTimeRange(v)`) — query-param paths that route through `ParseTimeRange` rather than `ParseTimeRangeStrict` |
| `time_range.go:314` | `ParseTimeRange(map)` adapter (`return ToTimeRange(startStr + "/" + endStr)`) — map-shaped query/body input |
| `history.go:39` | HistoricTime fallback string parse (`tr := ToTimeRange(s)`) — Datastream history time |

All four paths inherit the silent-discard. Severity assessment in §10
retains **P3** because the parent fix already covers the high-traffic
JSON request-body array/object branches; remaining surface is the
slash-delimited string form (less canonical) plus `history.go`.

**Live re-verification:** not captured this round. POST of a
Datastream / Observation requires authenticated admin endpoints not
exposed on `csapi-go-upstream`. The defect is structural — the four
delegation sites + the three legacy `t, _ := time.Parse` sites are
all in source. Parent #18's pre-`c2ab201` live evidence
([`../evidence/issue-018/live-test-2026-04-30.md`](../evidence/issue-018/live-test-2026-04-30.md))
demonstrated the year-0001 corruption shape on the same code path;
`c2ab201` fixed two of three branches but left `ToTimeRange` itself
untouched.

## 2. Static evidence

Source: [`../evidence/issue-018/static-analysis-2026-04-30.md`](../evidence/issue-018/static-analysis-2026-04-30.md)
(refreshed in §1 above).

`internal/model/common_shared/time_range.go` `ToTimeRange`
(lines 129-170 on `df6da0d`):

```go
func ToTimeRange(timeValue string) TimeRange {
    if timeValue == "latest" {
        return TimeRange{Latest: true}
    }
    if timeValue == "now" { /* … */ }
    if timeValue == "../.." || timeValue == ".." {
        return TimeRange{}
    }

    parts := strings.Split(timeValue, "/")

    if len(parts) == 2 {
        var startTime, endTime *time.Time

        if parts[0] != "" && parts[0] != ".." {
            t, _ := time.Parse(time.RFC3339, parts[0])    // legacy #1 — err discarded, &t taken regardless
            startTime = &t
        }

        if parts[1] != "" && parts[1] != ".." {
            t, _ := time.Parse(time.RFC3339, parts[1])    // legacy #2 — err discarded, &t taken regardless
            endTime = &t
        }

        return TimeRange{Start: startTime, End: endTime}
    }

    // Single value (no slash): treat as start time only
    if len(parts) == 1 && parts[0] != "" && parts[0] != ".." {
        t, _ := time.Parse(time.RFC3339, parts[0])        // legacy #3 — err discarded, &t taken regardless
        return TimeRange{Start: &t, End: nil}
    }

    return TimeRange{Start: nil, End: nil}
}
```

When `time.Parse` fails, `t` is the zero `time.Time` value
(year 0001). The legacy pattern takes `&t` regardless, so the
returned `TimeRange` carries a non-nil pointer to year-0001 instead
of `nil`. This is **strictly worse than nil**: downstream code that
checks `if tr.Start == nil` will not catch the corruption, only code
that additionally checks `tr.Start.IsZero()` will.

`internal/model/common_shared/time_range.go:119` (UnmarshalJSON
string-form branch):

```go
// try string form
var s string
if err := json.Unmarshal(b, &s); err == nil {
    *tr = ToTimeRange(s)         // <-- delegation site
    return nil
}
```

`internal/model/common_shared/history.go:39`:

```go
// fallback: interpret string as a range (start/end)
tr := ToTimeRange(s)
ht.Range = &tr
```

`time_range.go:298` (`ParseTimeRange(string)` adapter):

```go
case string:
    return ToTimeRange(v)
```

`time_range.go:314` (`ParseTimeRange(map)` adapter):

```go
case map[string]interface{}:
    startStr, _ := v["start"].(string)
    endStr, _ := v["end"].(string)
    return ToTimeRange(startStr + "/" + endStr)
```

A strict-parsed sibling, `toTimeRangeStrict`, already exists in the
same file (lines 215+) and is the basis of `ParseTimeRangeStrict`.
The defect is that `ToTimeRange` still exposes the silent-discard
channel via the four delegation sites above.

## 3. Live evidence

Not captured this round (auth-gated POST). Justification in §1.
Parent #18's pre-`c2ab201` evidence transitively applies — same file,
same parse pattern, same year-0001 sink.

## 4. Spec authority

Same spec posture as parent #18.

| Source | List entry it traces to | Used for |
|---|---|---|
| **RFC 7493** §3.4 (I-JSON time format) | "RFC 7493 — The I-JSON Message Format" under *IETF RFCs* | Direct binding: I-JSON time values must conform to RFC 3339. Silent acceptance of non-RFC-3339 input violates this. **Primary citation, same as #18.** |
| **OGC 23-001** — CSAPI Part 1 §"Time encoding" / `phenomenonTime` schema | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms `phenomenonTime` is RFC 3339, not free-form. |
| **OGC 23-002** — CSAPI Part 2 Datastream / Observation schemas | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Confirms TimeRange schema and the resources that embed it. |
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: malformed time string is malformed request syntax → 400, not silent acceptance with corrupted state. |

**Out-of-scope sources (not cited):** OAS 3.0.3, JSON Schema 2020-12.

## 5. Alternatives considered (internal-only)

### Option A — Inline-guard the three sites (recommended)

```go
if t, err := time.Parse(time.RFC3339, parts[0]); err == nil {
    startTime = &t
}
// (same shape for parts[1] and the single-value branch)
```

- Smallest diff. Three sites, mechanical change.
- No API surface change; `ToTimeRange(string) TimeRange` signature
  preserved.
- Behaviour shift: invalid input now produces `nil` pointer rather
  than year-0001 pointer. Aligns with the existing
  `ToTimeRangeFromSlice` shape (which already uses `if … err == nil`
  guards — see lines 187-201 of the same file).
- Downside: still silent — no 400 on the
  `UnmarshalJSON`-string-form path. But this matches the maintainer's
  established convention for `To*` helpers (lossy → strict variant
  exists separately).

### Option B — Promote `ToTimeRange` to `(TimeRange, error)` and migrate callers

- Eliminates the silent-discard channel structurally.
- Requires migrating four call-sites + 1 test (5 touch points).
- Returns 400 on the `UnmarshalJSON` string-form path (better UX).
- Risk: callers that currently rely on lossy behaviour (none found
  in `df6da0d`, but worth maintainer review).
- Larger diff.

### Option C — Replace internal calls with `toTimeRangeStrict`

- The strict-parsing infrastructure already exists
  (`toTimeRangeStrict`, line 215). Migrate the four delegation
  sites to call it instead of `ToTimeRange`, and bubble the error.
- Functionally equivalent to Option B but reuses existing code.
- Requires same call-site changes as Option B but no new function.
- Probably the cleanest long-term shape. Mention in §10.

**Lead with Option A.** Mention Options B/C as "if you'd prefer to
close the channel structurally, the strict-parse infrastructure
already exists — happy to follow up with a PR".

## 6. Recommended fix

**Option A.** Inline-guard three sites in `ToTimeRange`:

```diff
 if parts[0] != "" && parts[0] != ".." {
-    t, _ := time.Parse(time.RFC3339, parts[0])
-    startTime = &t
+    if t, err := time.Parse(time.RFC3339, parts[0]); err == nil {
+        startTime = &t
+    }
 }

 if parts[1] != "" && parts[1] != ".." {
-    t, _ := time.Parse(time.RFC3339, parts[1])
-    endTime = &t
+    if t, err := time.Parse(time.RFC3339, parts[1]); err == nil {
+        endTime = &t
+    }
 }

 // Single value (no slash): treat as start time only
 if len(parts) == 1 && parts[0] != "" && parts[0] != ".." {
-    t, _ := time.Parse(time.RFC3339, parts[0])
-    return TimeRange{Start: &t, End: nil}
+    if t, err := time.Parse(time.RFC3339, parts[0]); err == nil {
+        return TimeRange{Start: &t, End: nil}
+    }
+    return TimeRange{}
 }
```

Implementation surface: one function (`ToTimeRange`), three guards.
Aligns shape with neighbouring `ToTimeRangeFromSlice` which already
uses this pattern (lines 187-201).

## 7. Scope guard

What NOT to touch as part of this filing:

- No change to `UnmarshalJSON` array/object branches — parent fix
  (`c2ab201`) already strict-parses these.
- No change to `Observation.ResultTime` handling — already strict
  (cited in eval as justification only).
- No removal of `ToTimeRange`. Option B/C deferred to maintainer
  preference; do not include in primary recommendation.
- No new API surface (`ToTimeRangeStrict` already exists as
  `toTimeRangeStrict` / `ParseTimeRangeStrict`; no rename needed).
- No callers outside `time_range.go` and `history.go`. Audit (§1)
  confirms only those two files plus internal adapters.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#18` was the umbrella P2 finding for
  `TimeRange`-decode silent-discard. Closed by upstream `c2ab201`
  ("time range and better 400 …"). Issue #18's own scope deferred
  the `ToTimeRange` legacy pattern to a separate residual filing
  (this one) because at #18's filing time the array/object branches
  were the exposed primary surface.
- `c2ab201` strict-parsed `UnmarshalJSON` array+object and added
  `ParseTimeRangeStrict` infrastructure. It also added a new
  "single value" branch to `ToTimeRange` that uses the same legacy
  `t, _ := time.Parse` shape (legacy site #3) — confirms the
  legacy pattern is institutional, not a one-off.
- No fork-side patch for this defect; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Recommend inline-guard (A) or strict-rewrite (B/C)? | **Lead with A.** Smaller diff, maintainer-friendly, matches `ToTimeRangeFromSlice` shape. Mention B/C as structural alternatives. |
| Severity P3 vs P2? | **P3.** Parent fix covers high-traffic array/object decode. Remaining channel is slash-delimited string form (less canonical) + `history.go`. |
| Bundle `Observation.ResultTime` asymmetric-strictness note? | **No.** Cited as justification (proves silent-discard is a defect, not a deliberate API choice) — not a fix-scope item. |
| Live reproducer required? | **No.** Auth-gated; defect is structural; static + parent #18 live evidence sufficient. |
| Caller count? | **Four**, not two as backlog claimed. Documented in §1 caller-graph table. Filing scope unchanged (still narrow), but recommendation must mention all four delegation sites. |
| Cite eval/evidence paths? | **Yes**, validation-chain footer. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P3] ToTimeRange silent-discards parse errors → year-0001 pointer (residual of c2ab201 / issue #18)`

**Labels:** `bug`

---

### Context

`c2ab201` ("time range and better 400 …") strict-parsed
`TimeRange.UnmarshalJSON`'s array and object branches and introduced
`ParseTimeRangeStrict` for query-param validation. The legacy
`ToTimeRange(string) TimeRange` helper was not updated and still
discards parse errors via `t, _ := time.Parse(...); &t`, returning
a non-nil pointer to year-0001 (`0001-01-01T00:00:00Z`) on malformed
input.

`ToTimeRange` is reachable from four non-test call sites that route
JSON or query input into the helper — see "Caller graph" below.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

`internal/model/common_shared/time_range.go` `ToTimeRange` discards
`time.Parse` errors at three sites (lines 147, 152, 161 on `df6da0d`)
and returns a `TimeRange` with `Start` and/or `End` pointing to a
zero `time.Time` (year-0001) instead of `nil`. This is **strictly
worse than nil**: callers that test `if tr.Start == nil` do not
detect the corruption; only callers that additionally test
`tr.Start.IsZero()` do.

### Caller graph

```text
internal/model/common_shared/time_range.go:119   UnmarshalJSON string-form (*tr = ToTimeRange(s))
internal/model/common_shared/time_range.go:298   ParseTimeRange(string)       (return ToTimeRange(v))
internal/model/common_shared/time_range.go:314   ParseTimeRange(map)          (return ToTimeRange(startStr + "/" + endStr))
internal/model/common_shared/history.go:39       HistoricTime fallback string (tr := ToTimeRange(s))
```

The `UnmarshalJSON` string-form path means a JSON request body of
`{"phenomenonTime":"junk/junk"}` decodes to a TimeRange with
year-0001 endpoints, not an error.

### Static evidence

`internal/model/common_shared/time_range.go` `ToTimeRange` (excerpt,
HEAD `df6da0d`):

```go
func ToTimeRange(timeValue string) TimeRange {
    // … special-value branches …

    parts := strings.Split(timeValue, "/")

    if len(parts) == 2 {
        var startTime, endTime *time.Time

        if parts[0] != "" && parts[0] != ".." {
            t, _ := time.Parse(time.RFC3339, parts[0])    // <-- silent
            startTime = &t                                //     pointer to year-0001 on err
        }
        if parts[1] != "" && parts[1] != ".." {
            t, _ := time.Parse(time.RFC3339, parts[1])    // <-- silent
            endTime = &t
        }
        return TimeRange{Start: startTime, End: endTime}
    }

    if len(parts) == 1 && parts[0] != "" && parts[0] != ".." {
        t, _ := time.Parse(time.RFC3339, parts[0])        // <-- silent
        return TimeRange{Start: &t, End: nil}
    }
    return TimeRange{Start: nil, End: nil}
}
```

For comparison, the sibling `ToTimeRangeFromSlice` already uses the
correct guard shape:

```go
if t, err := time.Parse(time.RFC3339, parts[0]); err == nil {
    startTime = &t
}
```

`UnmarshalJSON` string-form branch (line 119):

```go
var s string
if err := json.Unmarshal(b, &s); err == nil {
    *tr = ToTimeRange(s)
    return nil
}
```

### Live evidence

Not included in this filing — POST sequence requires authenticated
admin endpoints. The defect is structural (three `_ :=` discards in
source, four reachable delegation sites). Parent issue #18's
pre-`c2ab201` live evidence demonstrated the year-0001 corruption on
the same code path.

### Recommended fix

Inline-guard the three sites, mirroring the existing
`ToTimeRangeFromSlice` shape:

```diff
 if parts[0] != "" && parts[0] != ".." {
-    t, _ := time.Parse(time.RFC3339, parts[0])
-    startTime = &t
+    if t, err := time.Parse(time.RFC3339, parts[0]); err == nil {
+        startTime = &t
+    }
 }
```

(and analogous for `parts[1]` and the single-value branch).

Implementation surface: one function, three guards in
`internal/model/common_shared/time_range.go`. No API change. No
caller migration needed.

If you'd prefer to close the channel structurally rather than just
make the failure mode `nil` instead of year-0001, the strict-parse
infrastructure already exists (`toTimeRangeStrict` /
`ParseTimeRangeStrict`); migrating the four delegation sites to use
it would return 400 on malformed input rather than silently
discarding. Happy to follow up with a separate PR if that shape is
preferred.

### Spec authority

- **RFC 7493** §3.4 (I-JSON) — time values must conform to RFC 3339.
  Silent acceptance of non-RFC-3339 input violates this. Same SHOULD
  invoked for parent issue #18.
- **OGC 23-001** CSAPI Part 1 — `phenomenonTime` is RFC 3339.
- **RFC 9110** §15.5.1 — malformed request syntax warrants 400; in
  Option A's shape the input is silently dropped to nil rather than
  rejected, which is a softer landing but still defensible
  given the maintainer's established `To*` helper convention.

### Severity

**P3** — narrow residual exposure. Parent fix `c2ab201` already
covers the high-traffic JSON request-body array/object branches;
remaining surface is the slash-delimited string form (less canonical
client shape) plus the `history.go` HistoricTime fallback.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-018.md`](../issue-evaluations/issue-018.md) §"Adjacent finding"
- Evidence (static): [`docs/research/evidence/issue-018/static-analysis-2026-04-30.md`](../evidence/issue-018/static-analysis-2026-04-30.md)
- Evidence (live, parent): [`docs/research/evidence/issue-018/live-test-2026-04-30.md`](../evidence/issue-018/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-018/spec-authority-2026-04-30.md`](../evidence/issue-018/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-06-totimerange-year-0001-silent-discard.md`](../upstream-issues/plan-06-totimerange-year-0001-silent-discard.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #9
