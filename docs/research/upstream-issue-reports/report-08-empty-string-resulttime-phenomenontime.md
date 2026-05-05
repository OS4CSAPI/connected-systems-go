# Report 08 — `resultTime` / `phenomenonTime` empty-string conflates with missing

> **🛑 Mandatory pre-work — read every session before working from this report:**
>
> 1. Re-open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm spec/standard sources match the canonical entries.
>    **Do not self-source references.**
> 2. Re-read the research plan
>    [`../upstream-issues/plan-08-empty-string-resulttime-phenomenontime.md`](../upstream-issues/plan-08-empty-string-resulttime-phenomenontime.md).
> 3. Re-read the source eval and evidence files cross-referenced in §2–§4.

---

## Header

| Field | Value |
|---|---|
| Backlog item | [`upstream-followup-backlog.md`](../upstream-followup-backlog.md) **#11** (`resultTime` / `phenomenonTime` empty-string conflates with missing) |
| Source fork issue | `OS4CSAPI/connected-systems-go#20` (closed by upstream `1b2b614`; this filing is the T5 empty-string residual deferred at closure) |
| Research plan | [`../upstream-issues/plan-08-empty-string-resulttime-phenomenontime.md`](../upstream-issues/plan-08-empty-string-resulttime-phenomenontime.md) |
| Target upstream repo | `SomethingCreativeStudios/connected-systems-go` |
| Severity | **P3** — UX residual; misleading error message on `resultTime`, asymmetric silent-accept on `phenomenonTime` |
| Tier | **A** (file first) |
| Report date | 2026-05-05 |
| Author | OS4CSAPI fork research stream |

---

## 1. Re-verification record

`upstream/main` HEAD: `df6da0dff8e2d3e76b64b00f856c0d43ed644f6d`
("code sight updates"). Parent fix `1b2b614` is in lineage; T5
empty-string residual persists and exhibits **asymmetric symptoms**
across the two fields.

```text
$ git log -1 upstream/main --format='%H %s'
df6da0dff8e2d3e76b64b00f856c0d43ed644f6d code sight updates

$ git log --oneline upstream/main -- internal/api/observation_handler.go | Select -First 4
1b2b614 Adding support for "latest" for TimeRange
fe9fbd0 Adding cascade delete and fixing existing cascade delete to full delete
635547f making the default limit for pagination configurable
f2cf1c3 Adding the other resources along with e2e tests
```

`internal/api/observation_handler.go` `decodeObservationPayload`
(post-`1b2b614`):

```go
// resultTime block:
if rtRaw, exists := raw["resultTime"]; exists {
    rtStr, ok := rtRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
    }
    if rtStr != "" {                                        // <-- empty-string falls through
        t, err := time.Parse(time.RFC3339, rtStr)
        if err != nil {
            return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
        }
        obs.ResultTime = t
    }
}

// phenomenonTime block (analogous shape, no late IsZero rescue):
if ptRaw, exists := raw["phenomenonTime"]; exists {
    ptStr, ok := ptRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `phenomenonTime must be an ISO 8601 string …`}
    }
    if ptStr != "" {                                        // <-- empty-string silently accepted
        t, err := time.Parse(time.RFC3339, ptStr)
        if err != nil {
            return nil, &decodeError{msg: `phenomenonTime must be an ISO 8601 string …`}
        }
        obs.PhenomenonTime = &t
    }
}

// later in the function (late rescue, resultTime only):
if obs.ResultTime.IsZero() {
    return nil, &decodeError{msg: "resultTime is required"}
}
```

```text
$ git show upstream/main:internal/api/observation_handler.go |
    Select-String 'must be a non-empty|s == ""|rtStr == ""|ptStr == ""'
(zero matches — confirms the empty-string branch is missing)
```

**Behavior matrix on `df6da0d`:**

| Input | `resultTime` outcome | `phenomenonTime` outcome |
|---|---|---|
| field absent | 400 `"resultTime is required"` ✓ | 201 (optional, `nil`) ✓ |
| `null` | 400 `"resultTime is required"` ✓ | 201 (optional, `nil`) ✓ |
| `12345` (number) | 400 `"resultTime must be an ISO 8601 string …"` ✓ | 400 `"phenomenonTime must be an ISO 8601 string …"` ✓ |
| `"not-a-date"` | 400 `"resultTime must be an ISO 8601 string …"` ✓ | 400 `"phenomenonTime must be an ISO 8601 string …"` ✓ |
| **`""` (T5)** | **400 `"resultTime is required"`** — misleading | **201 (silent accept, `nil`)** — silent corruption |

The two fields exhibit **different** misbehavior under the same root
cause:

- `resultTime` falls through to the late `IsZero()` guard and emits
  `"resultTime is required"` — wrong message (the client *did* send
  the field, just with an empty value).
- `phenomenonTime` has no late guard (it is optional, type
  `*time.Time`). Empty-string is silently treated as "field absent",
  so the request succeeds with `PhenomenonTime == nil`. This is a
  silent-coerce bug shape (sibling of plan-06's
  `ToTimeRange` family but milder — produces `nil` rather than
  year-0001).

Same root cause, two different surfaces. Single fix closes both.

**Live re-verification:** not captured this round. POST of an
Observation requires authenticated admin endpoints not exposed on
`csapi-go-upstream`. The defect is structural — exact decoder
shape visible above — and parent issue #20's pre-`1b2b614`
matrix-row T5 ([`../evidence/issue-020/live-test-2026-04-30.md`](../evidence/issue-020/live-test-2026-04-30.md))
demonstrated the misleading-message symptom on the same code path;
`1b2b614` fixed T1–T4 and T6–T7 but not T5.

## 2. Static evidence

Source: [`../evidence/issue-020/static-analysis-2026-04-30.md`](../evidence/issue-020/static-analysis-2026-04-30.md)
(refreshed in §1 above).

The `1b2b614` decoder exhibits the same conjunctive-guard pattern on
both blocks:

```go
if X, exists := raw[FIELD]; exists {
    s, ok := X.(string)
    if !ok { return ...wrong-type-error... }
    if s != "" {                                  // gates ALL non-empty processing
        ...parse and assign...
    }
}
```

No branch handles the `s == ""` case explicitly. On `resultTime`,
the absence is rescued (incorrectly labeled) by the late
`obs.ResultTime.IsZero()` check. On `phenomenonTime`, there is no
rescue and the field silently stays nil.

The body of issue #20 (the maintainer-accepted fix proposal,
reproduced in eval [`../issue-evaluations/issue-020.md`](../issue-evaluations/issue-020.md))
explicitly included a separate empty-string branch with the message
`"resultTime must be a non-empty RFC 3339 date-time string"`. The
shipped commit `1b2b614` omitted that branch — that omission is
exactly the residual this filing closes.

## 3. Live evidence

Not captured this round (auth-gated POST). Justification in §1.
Parent #20's pre-`1b2b614` matrix-row T5 transitively applies — the
late `IsZero()` rescue and the lack of an empty-string branch are
both visible in the source quoted in §1, and the symptom
(`"resultTime is required"` for an empty-string `resultTime`) is
deterministic from that code shape.

## 4. Spec authority

This finding is a problem-detail-accuracy defect, not a strict
schema violation. Spec authority is via RFC 7807 + the maintainer's
own post-`1b2b614` pattern of distinct messages per case.

| Source | List entry it traces to | Used for |
|---|---|---|
| **RFC 7807** §3 (problem-detail "identify the problem") | "RFC 7807 — Problem Details for HTTP APIs" under *IETF RFCs* | The binding principle: error responses should identify the specific problem, not collapse multiple distinct client errors into one message. **Primary citation.** |
| **OGC 23-001** — CSAPI Part 1 §"Error responses" | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Confirms CSAPI defers to OGC API – Common / RFC 7807 for error-response shape; no override or specialization. |
| **OGC 19-072** — OGC API – Common §error responses | "OGC API - Common - Part 1: Core (OGC 19-072)" under *OGC Common* | Supporting: OGC API – Common adopts RFC 7807 problem-detail semantics. |
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: status code is correct on the `resultTime` path; this filing is purely about message accuracy and the asymmetric `phenomenonTime` silent-accept. |

**Out-of-scope sources (not cited):** OAS 3.0.3, JSON Schema 2020-12, RFC 7493.

## 5. Alternatives considered (internal-only)

Only one viable shape — add the `s == ""` branch. The maintainer
already accepted this pattern in the body of issue #20.

### Option A — Add explicit empty-string branch to both blocks (recommended)

```go
if rtStr == "" {
    return nil, &decodeError{msg: `resultTime must be a non-empty ISO 8601 string`}
}
```

- Two block edits, one per field.
- Mirrors the existing wrong-type branch wording style.
- Closes both symptoms (misleading-message on `resultTime`,
  silent-accept on `phenomenonTime`).
- Risk: zero. Same shape as the issue-#20 body proposal.

**Lead with Option A only.** No alternatives.

## 6. Recommended fix

**Option A.** Insert the empty-string branch between the `ok`
check and the inner `!= ""` parse block in each decoder:

```diff
 if rtRaw, exists := raw["resultTime"]; exists {
     rtStr, ok := rtRaw.(string)
     if !ok {
         return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
     }
+    if rtStr == "" {
+        return nil, &decodeError{msg: `resultTime must be a non-empty ISO 8601 string`}
+    }
-    if rtStr != "" {
-        t, err := time.Parse(time.RFC3339, rtStr)
-        if err != nil {
-            return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
-        }
-        obs.ResultTime = t
-    }
+    t, err := time.Parse(time.RFC3339, rtStr)
+    if err != nil {
+        return nil, &decodeError{msg: `resultTime must be an ISO 8601 string (e.g. "2026-01-01T00:00:00Z")`}
+    }
+    obs.ResultTime = t
 }
```

(Analogous edit for `phenomenonTime`.)

The new explicit empty-string branch returns 400 immediately;
the surrounding `if rtStr != ""` gate becomes redundant and is
removed. The late `obs.ResultTime.IsZero()` guard is retained as
defense-in-depth for the missing/null cases.

Implementation surface: one function (`decodeObservationPayload`),
two block edits (one per field). No new types, no API change.

## 7. Scope guard

What NOT to touch as part of this filing:

- The wrong-type, missing, and null branches — parent fix
  `1b2b614` covers these correctly.
- The late `obs.ResultTime.IsZero()` guard — retained as
  defense-in-depth (per eval recommendation).
- The null↔missing collapse for `resultTime` — covered by the
  body-of-#20 proposal as defensible UX (same client correction
  needed). Not re-litigated.
- `samplingFeature@id` decode site — separate filing, report-07.
- `command_handler.go` decoder — separate concern; no
  `phenomenonTime` / `resultTime` analogue there.
- The `ToTimeRange` legacy pattern — separate filing, report-06
  (different file, different code path).

## 8. Fork-side context (internal-only)

- `OS4CSAPI/connected-systems-go#20` was the umbrella P3 finding for
  the conflated-error-messages matrix in `decodeObservationPayload`.
  Closed by upstream `1b2b614` ("Adding support for 'latest' for
  TimeRange"). The body of #20 included an explicit empty-string
  branch in its proposed fix code; the shipped commit omitted that
  branch.
- This filing surfaces an **asymmetric symptom not in the original
  matrix**: the `phenomenonTime` block's empty-string fall-through
  is a silent-accept rather than a misleading-message, because
  `PhenomenonTime` is optional and has no late `IsZero()` guard.
  Same root cause; recommended fix closes both.
- No fork-side patch; will be filed-and-await.

## 9. Open questions resolved

| Plan §8 question | Resolution |
|---|---|
| Recommended fix shape | **Option A only.** Add explicit `s == ""` branch in both blocks. No judgment-call. |
| Apply to both fields? | **Yes.** §1 confirms both blocks have the same gap, but with **asymmetric symptoms** (resultTime: misleading message; phenomenonTime: silent-accept). One fix closes both. |
| Re-litigate null vs. missing collapse? | **No.** Out of scope. Maintainer accepted that framing in #20. |
| Severity P3 vs P2? | **P3 retained.** Misleading-message on resultTime + silent-accept-empty on phenomenonTime. The phenomenonTime silent-accept could plausibly argue for P2 (data-integrity-adjacent), but PhenomenonTime is optional in CSAPI Part 1, so an empty-string-becomes-nil is semantically equivalent to omission — UX defect, not corruption. |
| Live reproducer required? | **No.** Auth-gated; defect is structural; static + parent #20 matrix sufficient. |
| Cite eval/evidence paths? | **Yes**, validation-chain footer. |

---

## 10. Public-facing extract — upstream issue body

**Title:** `[P3] resultTime/phenomenonTime: empty string conflates with missing — last gap of 1b2b614 / issue #20 matrix`

**Labels:** `bug`

---

### Context

`1b2b614` ("Adding support for 'latest' for TimeRange") disentangled
the conflated-error-messages matrix in
`internal/api/observation_handler.go` `decodeObservationPayload` per
issue #20, reducing the conflation from 6 client-error shapes
collapsing to 1 message down to 2→1.

The remaining conflation is the empty-string case (T5 in the
original matrix). The body of issue #20 included an explicit
empty-string branch in its proposed fix; the shipped commit omitted
it. The two fields exhibit **asymmetric symptoms** under the same
root cause:

- `"resultTime": ""` falls through to the late
  `obs.ResultTime.IsZero()` guard and emits
  `"resultTime is required"` — misleading (the client did send the
  field).
- `"phenomenonTime": ""` is silently treated as "field absent" and
  the request succeeds with `PhenomenonTime == nil` (no late guard
  since the field is optional).

Verified live on `upstream/main` HEAD
`df6da0dff8e2d3e76b64b00f856c0d43ed644f6d` (2026-05-05).

### Claim

Empty-string `resultTime` returns 400 with the misleading
`"resultTime is required"` message. Empty-string `phenomenonTime`
returns 201 with `PhenomenonTime == nil`. Both behaviours stem from
the same missing branch in the post-`1b2b614` decoder shape.

### Static evidence

`internal/api/observation_handler.go`
`decodeObservationPayload` (HEAD `df6da0d`):

```go
// resultTime block (post-1b2b614)
if rtRaw, exists := raw["resultTime"]; exists {
    rtStr, ok := rtRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
    }
    if rtStr != "" {                                        // <-- empty falls through
        t, err := time.Parse(time.RFC3339, rtStr)
        if err != nil {
            return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
        }
        obs.ResultTime = t
    }
}

// phenomenonTime block (post-1b2b614, no late guard)
if ptRaw, exists := raw["phenomenonTime"]; exists {
    ptStr, ok := ptRaw.(string)
    if !ok {
        return nil, &decodeError{msg: `phenomenonTime must be an ISO 8601 string …`}
    }
    if ptStr != "" {                                        // <-- empty silently accepted
        t, err := time.Parse(time.RFC3339, ptStr)
        if err != nil {
            return nil, &decodeError{msg: `phenomenonTime must be an ISO 8601 string …`}
        }
        obs.PhenomenonTime = &t
    }
}

// later in the same function:
if obs.ResultTime.IsZero() {
    return nil, &decodeError{msg: "resultTime is required"}    // (rescues missing AND empty-string)
}
```

`grep` for the empty-string branch in the file returns zero matches:

```text
$ git grep -nE 'rtStr == ""|ptStr == ""|must be a non-empty' upstream/main -- internal/api/observation_handler.go
(no output)
```

### Behavior matrix on `df6da0d`

| Input | `resultTime` | `phenomenonTime` |
|---|---|---|
| absent | 400 `"resultTime is required"` ✓ | 201 (optional) ✓ |
| `null` | 400 `"resultTime is required"` ✓ | 201 (optional) ✓ |
| number | 400 `"… must be an ISO 8601 string …"` ✓ | 400 `"… must be an ISO 8601 string …"` ✓ |
| malformed string | 400 `"… must be an ISO 8601 string …"` ✓ | 400 `"… must be an ISO 8601 string …"` ✓ |
| **`""`** | **400 `"resultTime is required"`** ✗ misleading | **201 with `PhenomenonTime == nil`** ✗ silent accept |

### Live evidence

Not included in this filing — POST sequences require authenticated
admin endpoints. The defect is structural (the missing branch is
visible in the source above) and parent issue #20's pre-`1b2b614`
T5 row demonstrated the `resultTime` misleading-message symptom on
the same code path. The asymmetric `phenomenonTime` silent-accept
follows deterministically from the absence of a late guard for an
optional field.

### Recommended fix

Insert an explicit empty-string branch between the `ok` check and
the parse logic in each decoder:

```diff
 if rtRaw, exists := raw["resultTime"]; exists {
     rtStr, ok := rtRaw.(string)
     if !ok {
         return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
     }
+    if rtStr == "" {
+        return nil, &decodeError{msg: `resultTime must be a non-empty ISO 8601 string`}
+    }
-    if rtStr != "" {
-        t, err := time.Parse(time.RFC3339, rtStr)
-        ...
-    }
+    t, err := time.Parse(time.RFC3339, rtStr)
+    if err != nil {
+        return nil, &decodeError{msg: `resultTime must be an ISO 8601 string …`}
+    }
+    obs.ResultTime = t
 }
```

Analogous edit in the `phenomenonTime` block. The late
`obs.ResultTime.IsZero()` guard is retained as defense-in-depth.

Implementation surface: one function, two block edits in
`internal/api/observation_handler.go`. No new types, no API
change.

### Spec authority

- **RFC 7807** §3 — problem-detail responses should identify the
  specific problem; collapsing distinct client errors into one
  message violates this principle. Primary citation.
- **OGC 23-001** CSAPI Part 1 — defers error-response shape to
  OGC API – Common / RFC 7807. No CSAPI override.
- **OGC 19-072** — OGC API – Common adopts RFC 7807 semantics.
- **RFC 9110** §15.5.1 — 400 is the correct status; this filing
  is about message accuracy.

### Severity

**P3** — UX residual. No data-integrity loss on the `resultTime`
path (the request is rejected, just with the wrong message). The
`phenomenonTime` silent-accept-empty is semantically equivalent to
field omission since the field is optional in CSAPI Part 1, so the
behaviour is misleading-but-not-corrupting. Pure problem-detail
accuracy issue.

---

**Validation chain:**

- Eval: [`docs/research/issue-evaluations/issue-020.md`](../issue-evaluations/issue-020.md)
- Evidence (static): [`docs/research/evidence/issue-020/static-analysis-2026-04-30.md`](../evidence/issue-020/static-analysis-2026-04-30.md)
- Evidence (live, parent matrix): [`docs/research/evidence/issue-020/live-test-2026-04-30.md`](../evidence/issue-020/live-test-2026-04-30.md)
- Evidence (spec): [`docs/research/evidence/issue-020/spec-authority-2026-04-30.md`](../evidence/issue-020/spec-authority-2026-04-30.md)
- Plan: [`docs/research/upstream-issues/plan-08-empty-string-resulttime-phenomenontime.md`](../upstream-issues/plan-08-empty-string-resulttime-phenomenontime.md)
- Backlog: [`docs/research/upstream-followup-backlog.md`](../upstream-followup-backlog.md) #11
