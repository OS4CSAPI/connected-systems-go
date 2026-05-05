# Report 07 — `samplingFeature@id` retains silent-drop type assertion

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-07-samplingfeature-id-silent-drop.md`](../upstream-issues/plan-07-samplingfeature-id-silent-drop.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#10** (`samplingFeature@id` retains silent-drop type assertion) |
| Source fork issue | `OS4CSAPI/connected-systems-go#19` (closed by upstream `1b2b614`; this filing is the "adjacent finding" residual deferred at closure) |
| Research plan | [`../upstream-issues/plan-07-samplingfeature-id-silent-drop.md`](../upstream-issues/plan-07-samplingfeature-id-silent-drop.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P3** — narrow blast radius (no repository default-fill, so silent drop yields a null/absent field on GET, not a plausibly-correct substitution) |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Parent fix `1b2b614` is in lineage; defect
persists at two sites.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git log --oneline upstream/main | Select-String 1b2b614
1b2b614 Adding support for "latest" for TimeRange
```

**Bundle audit — all `raw["…"].(string); ok` patterns in `internal/api/`:**

```text
$ git grep -nE 'raw\["[^"]+"\]\.\(string\)' upstream/main -- internal/api/
internal/api/command_handler.go:217   if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
internal/api/command_handler.go:221   if sender, ok := raw["sender"].(string); ok {
internal/api/command_handler.go:225   if status, ok := raw["currentStatus"].(string); ok && status != "" {
internal/api/command_handler.go:229   if issueTimeStr, ok := raw["issueTime"].(string); ok && issueTimeStr != "" {
internal/api/observation_handler.go:225   if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
```

The `samplingFeature@id` site appears **twice** — once in
`observation_handler.go` (the original eval target) and once in
`command_handler.go` (the immediate-sibling decoder). Both files are
in the same package and use the same decode-from-`raw map[string]any`
idiom. Per plan §8 Q2, bundling these two sites — same field name,
same shape, same package, same parent-fix author. The other
`command_handler.go` legacy patterns (`sender`, `currentStatus`,
`issueTime`) are different fields and are out of scope here.

```text
$ git show upstream/main:internal/api/observation_handler.go |
    Select-String 'samplingFeature@id|phenomenonTime|resultTime' -Context 4,4

# observation_handler.go:225 (LEGACY — silent drop)
    obs := &domains.Observation{}

>   if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
        obs.SamplingFeatureID = &sfID
    }

# observation_handler.go:238 (POST-1b2b614 — explicit error)
>   if rtRaw, exists := raw["resultTime"]; exists {
        rtStr, ok := rtRaw.(string)
        if !ok {
>           return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
        }
        if rtStr != "" {
            t, err := time.Parse(time.RFC3339, rtStr)
            if err != nil {
>               return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
            }
            obs.ResultTime = t
        }
    }

# observation_handler.go:252 (POST-1b2b614 — explicit error)
>   if ptRaw, exists := raw["phenomenonTime"]; exists {
        ptStr, ok := ptRaw.(string)
        if !ok {
>           return nil, &decodeError{msg: `phenomenonTime must be an ISO 8601 string …`}
        }
        …
    }
```

Asymmetry confirmed: two of three decoder blocks in
`decodeObservationPayload` use the
`present && nil-check && type-assert-with-explicit-error` pattern;
the `samplingFeature@id` block does not.

```text
$ git show upstream/main:internal/api/command_handler.go | Select -Skip 200 -First 30

func decodeCommandPayload(r *http.Request) (*domains.Command, error) {
    var raw map[string]any
    if err := json.NewDecoder(r.Body).Decode(&raw); err != nil {
        return nil, err
    }

    cmd := &domains.Command{}

    if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {   <-- legacy
        cmd.SamplingFeatureID = &sfID
    }
    …
}
```

Same shape, same defect.

**Live re-verification:** not captured this round. POST of an
Observation requires authenticated admin endpoints not exposed on
`csapi-go-upstream`. The defect is structural — both call sites are
in source — and parent issue #19's pre-`1b2b614` live evidence
([`../evidence/issue-019/live-test-2026-04-30.md`](../evidence/issue-019/live-test-2026-04-30.md))
demonstrated the silent-drop behavior on the same decoder family;
`1b2b614` fixed two of three blocks in `observation_handler.go` and
none of the matching blocks in `command_handler.go`.

## 2. Static evidence

Source: [`../evidence/issue-019/static-analysis-2026-04-30.md`](../evidence/issue-019/static-analysis-2026-04-30.md)
(refreshed in §1 above).

`internal/api/observation_handler.go:225` — **legacy pattern**:

```go
if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
    obs.SamplingFeatureID = &sfID
}
```

`internal/api/observation_handler.go:238-263` —
**post-`1b2b614` pattern** (the precedent):

```go
if rtRaw, exists := raw["resultTime"]; exists {
    rtStr, ok := rtRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
    }
    if rtStr != "" {
        t, err := time.Parse(time.RFC3339, rtStr)
        if err != nil {
            return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
        }
        obs.ResultTime = t
    }
}
// (analogous block for phenomenonTime)
```

`internal/api/command_handler.go:217` — **legacy pattern (sibling)**:

```go
if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
    cmd.SamplingFeatureID = &sfID
}
```

For both legacy sites, when the client sends a JSON value of any
non-string type (`number`, `bool`, `array`, `object`, `null`), the
type assertion fails (`ok == false`), the block is skipped silently,
and the parent record is created with `SamplingFeatureID == nil`.
There is no repository default-fill, so subsequent GET responses show
`samplingFeature@id` as null/absent — the corruption is
client-visible but not server-flagged.

## 3. Live evidence

Not captured this round (auth-gated POST). Justification in §1.
Parent #19's pre-`1b2b614` evidence transitively applies — same
file (`observation_handler.go`), same `raw[...].(string); ok`
pattern, same silent-drop sink.

## 4. Spec authority

Same spec posture as parent #19.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-001** — CSAPI Part 1 §5.1 / Observation schema | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms `samplingFeature@id` is typed as a string ID reference in the canonical Observation schema. **Primary citation** — establishes that a JSON number / object / boolean is wrong-typed input. |
| **OGC 23-002** — CSAPI Part 2 Observation / Command schemas | "OGC API - Connected Systems - Part 2: Dynamic Data" / "Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Corroborates the string-typed schema for `samplingFeature@id` on both resource types. |
| **RFC 7493** §3.4 (I-JSON wrong-type rejection) | "RFC 7493 — The I-JSON Message Format" under *IETF RFCs* | Robustness rule: implementations should reject inputs that don't conform to the schema rather than silently coerce or drop. Same SHOULD invoked for parent issue #19. |
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: malformed request body → 400 with informative message, not silent acceptance. |

**Out-of-scope sources (not cited):** OAS 3.0.3, JSON Schema 2020-12, SWE Common.

## 5. Alternatives considered (internal-only)

Only one viable shape — mirror `1b2b614`'s pattern byte-for-byte.

### Option A — Apply parent-fix pattern to both `samplingFeature@id` sites (recommended)

```go
if sfRaw, exists := raw["samplingFeature@id"]; exists {
    sfID, ok := sfRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `samplingFeature@id must be a string`}
    }
    if sfID != "" {
        obs.SamplingFeatureID = &sfID    // (or cmd.SamplingFeatureID for command_handler)
    }
}
```

- One block-for-block replacement at each of two sites
  (`observation_handler.go:225`, `command_handler.go:217`).
- Mirrors `1b2b614` shape exactly; uses existing `decodeError` type.
- Behavior shift: numeric / object / boolean
  `samplingFeature@id` now produces 400 instead of 201-with-drop.
  String input round-trips unchanged. Empty-string input behaves
  the same as today (silent skip — preserves the maintainer's
  established convention for empty-string-as-absent).
- Risk: zero. Same shape the maintainer already accepted on the
  same file in `1b2b614`.

**Lead with Option A only.** No other shape considered.

## 6. Recommended fix

**Option A** — apply the `1b2b614` pattern to both
`samplingFeature@id` decode sites:

```diff
--- a/internal/api/observation_handler.go
+++ b/internal/api/observation_handler.go
-    if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
-        obs.SamplingFeatureID = &sfID
+    if sfRaw, exists := raw["samplingFeature@id"]; exists {
+        sfID, ok := sfRaw.(string)
+        if !ok {
+            return nil, &decodeError{msg: `samplingFeature@id must be a string`}
+        }
+        if sfID != "" {
+            obs.SamplingFeatureID = &sfID
+        }
     }
```

```diff
--- a/internal/api/command_handler.go
+++ b/internal/api/command_handler.go
-    if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
-        cmd.SamplingFeatureID = &sfID
+    if sfRaw, exists := raw["samplingFeature@id"]; exists {
+        sfID, ok := sfRaw.(string)
+        if !ok {
+            return nil, &decodeError{msg: `samplingFeature@id must be a string`}
+        }
+        if sfID != "" {
+            cmd.SamplingFeatureID = &sfID
+        }
     }
```

Implementation surface: two block edits across two files in the same
package. No new types, no API change, no schema change.

## 7. Scope guard

What NOT to touch as part of this filing:

- `phenomenonTime` and `resultTime` decode blocks in
  `observation_handler.go` — parent fix (`1b2b614`) already
  strict-checks these.
- `command_handler.go` legacy patterns for `sender`,
  `currentStatus`, `issueTime`, and `executionTime`. Each is a
  different field type with its own validation considerations
  (`sender` accepts empty; `currentStatus` is an enum with a
  separate concern; `issueTime` / `executionTime` are time-format
  parses that overlap with backlog item #9 / report-06's
  `ToTimeRange` family). **Do not bundle.** A separate hardening
  pass for `command_handler.go`'s decoder is appropriate — out of
  scope here.
- No change to repository default-fill behavior (none exists for
  `samplingFeature@id` and that's intentional — the absence is
  surfaced to the client via null in the GET response).
- No schema changes.
- No rename or relocation of `decodeObservationPayload` /
  `decodeCommandPayload`.

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#19` was the umbrella P2 finding for
  Observation decoder silent-drop. Closed by upstream `1b2b614`
  ("Adding support for 'latest' for TimeRange") which strict-checked
  `phenomenonTime` and `resultTime`. Issue #19's own scope deferred
  the `samplingFeature@id` block to a separate residual filing
  (this one).
- Bundle audit during re-verification surfaced the
  `command_handler.go` sibling site — not in the original eval but
  same field, same shape, same package. Bundled per plan §8 Q2.
- No fork-side patch; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Recommended fix shape | **Option A only.** Mirror `1b2b614` byte-for-byte. No judgment-call. |
| Bundle other ID-reference fields with the same pattern? | **Bundle the `command_handler.go:217` sibling site for `samplingFeature@id`** — same field, same shape, same package. Do not bundle the other `command_handler.go` legacy patterns (`sender`, `currentStatus`, `issueTime`, `executionTime`); different fields, different concerns. |
| Severity P3 vs P2? | **P3.** Narrower blast than parent #19: no repository default-fill, so the silent drop is client-visible via null GET response, not a plausibly-correct substitution. |
| Live reproducer required? | **No.** Auth-gated; defect is structural (two static call sites); static + parent #19 live evidence sufficient. |
| Cite eval/evidence paths? | **Yes**, validation-chain footer. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P3] samplingFeature@id silently dropped on wrong-typed input — residual of 1b2b614 / issue #19 (also affects command_handler.go)`

**Labels:** `bug`

---

### Context

`1b2b614` ("Adding support for 'latest' for TimeRange") applied a
`present && nil-check && type-assert-with-explicit-error` pattern to
the `phenomenonTime` and `resultTime` decode blocks in
`internal/api/observation_handler.go`'s `decodeObservationPayload`,
returning `decodeError` (→ HTTP 400) on wrong-typed input.

The third decoder block in the same function — `samplingFeature@id` —
was not updated and retains the legacy
`if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != ""`
pattern. The matching block in the immediate-sibling
`command_handler.go` `decodeCommandPayload` has the same legacy
shape. In both, a non-string client value silently fails the type
assertion and the field is dropped from the persisted record.

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

A JSON value of any non-string type for `samplingFeature@id` on
`POST /datastreams/{id}/observations` or
`POST /controlstreams/{id}/commands` is silently dropped: the row is
created with `SamplingFeatureID == nil` and the GET response shows
the field as null/absent. Affects two decoder sites:

```text
internal/api/observation_handler.go:225
internal/api/command_handler.go:217
```

### Static evidence

Asymmetry within `decodeObservationPayload` (HEAD `df6da0d`):

```go
// observation_handler.go:225 — LEGACY
if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
    obs.SamplingFeatureID = &sfID
}

// observation_handler.go:238 — POST-1b2b614
if rtRaw, exists := raw["resultTime"]; exists {
    rtStr, ok := rtRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
    }
    if rtStr != "" {
        t, err := time.Parse(time.RFC3339, rtStr)
        if err != nil {
            return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
        }
        obs.ResultTime = t
    }
}

// observation_handler.go:252 — POST-1b2b614 (analogous to resultTime)
```

Sibling `decodeCommandPayload` (HEAD `df6da0d`):

```go
// command_handler.go:217 — LEGACY
if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
    cmd.SamplingFeatureID = &sfID
}
```

Both legacy sites silently drop wrong-typed input. There is no
repository default-fill for `samplingFeature@id`, so the corruption
is visible to the client as null/absent in the GET response — but
no error is surfaced at write time.

### Live evidence

Not included in this filing — POST sequences require authenticated
admin endpoints. The defect is structural (two static call sites
with the legacy shape) and parent issue #19's pre-`1b2b614` live
evidence demonstrated the silent-drop behavior on the same decoder
family.

### Recommended fix

Apply `1b2b614`'s pattern to both `samplingFeature@id` decode
sites:

```diff
-    if sfID, ok := raw["samplingFeature@id"].(string); ok && sfID != "" {
-        obs.SamplingFeatureID = &sfID
+    if sfRaw, exists := raw["samplingFeature@id"]; exists {
+        sfID, ok := sfRaw.(string)
+        if !ok {
+            return nil, &decodeError{msg: `samplingFeature@id must be a string`}
+        }
+        if sfID != "" {
+            obs.SamplingFeatureID = &sfID
+        }
     }
```

(and analogous in `command_handler.go` with `cmd.SamplingFeatureID`).

Implementation surface: two block edits across two files
(`observation_handler.go`, `command_handler.go`), same package, same
shape as the parent fix. No new types, no API change, no schema
change.

### Spec authority

- **OGC 23-001** CSAPI Part 1 §5.1 / Observation schema —
  `samplingFeature@id` is typed as a string ID reference. Wrong-typed
  input is a contract violation.
- **OGC 23-002** CSAPI Part 2 — corroborates the string-typed schema
  on both Observation and Command resource types.
- **RFC 7493** §3.4 — I-JSON robustness: implementations should
  reject wrong-typed inputs rather than silently coerce or drop.
- **RFC 9110** §15.5.1 — 400 is the appropriate response for
  malformed request body content.

### Severity

**P3** — narrow blast radius. Unlike `phenomenonTime` /
`resultTime` (which parent #19 covered, and where a repository
default-fill could produce a plausibly-correct substitution),
`samplingFeature@id` has no default-fill, so the silent drop
surfaces directly to the client as null in the GET response.
Client-visible but server-unflagged.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-019.md`](../issue-evaluations/issue-019.md) §"Adjacent finding"
- Evidence (static): [`docs/research/evidence/issue-019/static-analysis-2026-04-30.md`](../evidence/issue-019/static-analysis-2026-04-30.md)
- Evidence (live, parent): [`docs/research/evidence/issue-019/live-test-2026-04-30.md`](../evidence/issue-019/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-019/spec-authority-2026-04-30.md`](../evidence/issue-019/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-07-samplingfeature-id-silent-drop.md`](../upstream-issues/plan-07-samplingfeature-id-silent-drop.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #10
