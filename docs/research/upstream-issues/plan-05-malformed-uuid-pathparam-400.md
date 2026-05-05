# Research Plan 05 — Malformed UUID path-param → 400 (currently 500)

> **🛑 Mandatory pre-work — read every session before touching this plan or
> its corresponding report:**
>
> 1. Open the curated authoritative-references list at
>    <https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>
>    and re-confirm the spec/standard sources cited below match the
>    canonical entries there. **Do not self-source references.** If a
>    citation is needed that is not in that list, stop and surface the
>    gap to the user before proceeding.
> 2. Re-read this entire plan file. Do not work from memory.
> 3. Re-read the source eval and evidence files linked in §3 before
>    drafting the report.

---

## 1. Backlog entry

From [`../upstream-followup-backlog.md`](../upstream-followup-backlog.md)
item **8**, copied verbatim:

> ### 8. Malformed UUID path-param → 400 (currently 500)
> - **Source:** Issue #17 closure.
> - **Category:** Defect (P3 — UX / contract).
> - **Summary:** `DELETE /datastreams/not-a-uuid` and similar produce SQLSTATE
>   `22P02` (`invalid_text_representation`) at the DB, which falls through
>   `errors.go`'s typed-sentinel switch (which only knows `ErrNotFound`,
>   `ErrHasChildren`, `isFKViolation` for 23503) to the default-500 arm. Should
>   return 400. `git grep -E 'uuid\.Parse|22P02|invalid_text_representation'
>   upstream/main -- internal/api/` returns zero matches. Cleanest fix is
>   `uuid.Parse(id)` at handler entry returning 400, short-circuiting the DB
>   round trip. Affects all path-param UUID handlers (DELETE, GET-by-id, PUT,
>   PATCH).
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **Severity note.** Backlog says **P3 — UX / contract**. The eval
> documented this as **Refinement 2** to the broader #17 thesis and
> noted both available fix shapes (handler-side `uuid.Parse` vs.
> repository-side `22P02` arm). Severity stays **P3**: it's a
> contract-correctness defect with no data-integrity or security
> impact (the malformed UUID never reaches the DB as a successful
> query — current behavior is 500, not silent success).

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#17` — closed by upstream commit(s)
  introducing `errors.go`'s typed-sentinel switch covering
  `ErrNotFound`, `ErrHasChildren`, and `isFKViolation` (PG `23503`).
- This filing is the **22P02 / malformed-UUID residual** identified
  in the eval as Refinement 2: the typed-sentinel switch landed but
  did not cover `22P02 invalid_text_representation`, so structurally
  invalid input still reaches the default-500 arm.
- No other fork issues bundle in. (Optional `23514` / check-constraint
  arm noted in eval Refinement 3 is explicitly out of scope per
  backlog.)

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-017.md`](../issue-evaluations/issue-017.md) | Full evaluation. "Refinements to status quo" §2 documents malformed UUIDs returning 204 *pre-#17-fix* (silent no-op). "Remediation review" §2 documents both fix shapes (handler-side `uuid.Parse` vs. repository-side `22P02` arm) and explicitly recommends the handler-side parse as cleaner (short-circuits before DB round-trip). |
| [`../evidence/issue-017/static-analysis-2026-04-30.md`](../evidence/issue-017/static-analysis-2026-04-30.md) | Source for the "zero matches" claim in `internal/api/` for `errors.As`/`pgconn`/`SQLSTATE`. Confirms no UUID-parse validation at handler entry. |
| [`../evidence/issue-017/live-test-2026-04-30.md`](../evidence/issue-017/live-test-2026-04-30.md) | Pre-#17-fix live evidence: `DELETE /datastreams/not-a-uuid` → 204 (silent no-op). **Post-#17-fix behavior must be re-captured during Step 4.2** to confirm the backlog-claimed transition from 204 → 500 (typed-sentinel switch passes the 22P02 through to the default-500 arm). |
| [`../evidence/issue-017/spec-authority-2026-04-30.md`](../evidence/issue-017/spec-authority-2026-04-30.md) | RFC 9110 §15.5.1 (400 Bad Request — "the server cannot or will not process the request due to something that is perceived to be a client error"). Direct binding for this finding. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer accepted issue #17 (a P2
status-code-classification umbrella) and shipped the typed-sentinel
switch in `errors.go`. Implications:

- Tone: **completing #17's coverage** for one additional SQLSTATE
  the typed-sentinel switch missed. Frame as "the switch you added
  in <commit> covers `23503`; here's the analogous handling for
  `22P02`."
- The maintainer's explicit choice to use a typed-sentinel switch
  (rather than handler-side input validation) is what makes the
  handler-side `uuid.Parse` recommendation a *judgment call*. Plan
  must respect both options in the recommendation.

## 5. Upstream commits relevant to the area

- The commit(s) that closed `#17` and added the typed-sentinel
  switch in `errors.go` — drafter must identify exactly via §6.
  Cite as the precedent for the fix shape and as the file the
  recommended fix would extend.
- Re-verification must capture `upstream/main` HEAD SHA at filing
  time and confirm both:
  (a) the typed-sentinel switch is in place, and
  (b) `22P02` is not handled.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Find the typed-sentinel switch and confirm coverage:
git show upstream/main:internal/api/errors.go
# (or whatever path the maintainer placed the helper at — also try internal/api/handlers/)
git ls-tree -r upstream/main | Select-String -Pattern 'errors\.go'

# Confirm 22P02 / invalid_text_representation are NOT handled anywhere in internal/api/:
git grep -nE 'uuid\.Parse|22P02|invalid_text_representation' upstream/main -- internal/api/

# Confirm 23503 IS handled (sanity, parent-fix in place):
git grep -nE '23503|isFKViolation|ErrHasChildren' upstream/main -- internal/api/

# Identify all path-param UUID handlers (DELETE, GET-by-id, PUT, PATCH) — the fix surface:
git grep -nE 'chi\.URLParam.*"id"|c\.Param\("id"\)|mux\.Vars' upstream/main -- internal/api/

# Recent history on errors.go (find the closing commit):
git log --oneline upstream/main -- internal/api/errors.go | Select-Object -First 10
```

```pwsh
# Live re-verification — no auth required:
curl.exe -s -o $null -w 'HTTP %{http_code}\n' -X DELETE 'https://129-80-248-53.sslip.io/csapi-go-head/datastreams/not-a-uuid'
curl.exe -s -o $null -w 'HTTP %{http_code}\n' -X DELETE 'https://129-80-248-53.sslip.io/csapi-go-head/systems/not-a-uuid'
curl.exe -s -o $null -w 'HTTP %{http_code}\n' 'https://129-80-248-53.sslip.io/csapi-go-head/datastreams/not-a-uuid'
curl.exe -s -o $null -w 'HTTP %{http_code}\n' 'https://129-80-248-53.sslip.io/csapi-go-head/systems/not-a-uuid'
# Optional: try a method that requires a body (PUT/PATCH) only if auth permits.
```

Expected on unchanged post-#17-fix state:
- `errors.go` typed-sentinel switch covers `ErrNotFound`,
  `ErrHasChildren`, `isFKViolation` (`23503`), default → 500.
- `git grep` for `uuid.Parse|22P02|invalid_text_representation`
  returns **zero matches** in `internal/api/`.
- Live `DELETE /<resource>/not-a-uuid` and
  `GET /<resource>/not-a-uuid` return **HTTP 500**.

If `22P02` is already handled or `uuid.Parse` is already at handler
entry on `upstream/main`, **stop drafting** — defect already fixed.

If live returns 400 instead of 500, the maintainer has fixed it
between eval and filing time. Confirm via `git log` and stop.

If live returns 204 (the pre-#17-fix behavior), the typed-sentinel
switch may not be in place, in which case the parent fix isn't
landed yet and this filing is premature — re-evaluate.

## 7. Spec-authority sources

This finding has direct spec binding via RFC 9110.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **RFC 9110** §15.5.1 (400 Bad Request) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Direct binding: "the server cannot or will not process the request due to something that is perceived to be a client error (e.g., malformed request syntax …)." A non-UUID path parameter on a UUID-typed resource identifier is unambiguously malformed request syntax. **Primary citation.** |
| **RFC 9110** §15.6.1 (500 Internal Server Error) | "RFC 9110 — HTTP Semantics" under *IETF RFCs* | Supporting: 500 is for unexpected server-side conditions, not foreseeable client input shapes. Reinforces that the current behavior mis-classifies the error class. |
| **OGC 19-072** §"Conformance class: Core" — error responses | "OGC API - Common - Part 1: Core (OGC 19-072)" under *OGC Common* | Supporting: OGC API – Common defers to RFC 9110 status-code semantics; no override or specialization for malformed identifiers. Confirms there is no spec-level reason cs-go should diverge. |
| **OGC 23-001** — CSAPI Part 1 path-parameter definitions for resource IDs | "OGC API - Connected Systems - Part 1: Core" under *OGC CSAPI Standards* | Supporting: confirms resource IDs are typed (UUID where applicable), so a non-UUID is a contract violation by the client, not an internal error. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — relevant only insofar as the OpenAPI definition (when
  it exists per backlog item 3 / plan-01) would declare the path
  parameter as `format: uuid`. Not a binding citation here; the
  RFC 9110 binding is direct.
- JSON Schema 2020-12 — not relevant.
- RFC 7493 — not relevant.

## 8. Open questions

1. **Recommend handler-side `uuid.Parse` or repository-side
   `22P02` arm?** Eval recommended handler-side as cleaner
   (short-circuits before DB round-trip; smaller blast radius;
   self-documenting at the handler boundary). Repository-side is
   smaller-diff and keeps all status mapping in one helper.
   **Tentative answer: lead with handler-side `uuid.Parse`,
   present repository-side as the smaller-diff alternative.**
   Explicitly defer to maintainer's preference.
2. **Where to place the handler-side parse?** Three options:
   (a) inline at every handler entry (12+ touch sites);
   (b) chi/mux middleware that validates `id` path-param;
   (c) a small helper called from each handler.
   **Tentative answer: defer to maintainer's existing convention
   for path-param parsing.** §6 audit will surface whether the
   codebase already has a pattern. If not, recommend (b)
   middleware as DRYest.
3. **Affects all four verbs (DELETE, GET-by-id, PUT, PATCH)?**
   Yes per backlog. Live re-verification covers DELETE and GET
   (no auth); PUT/PATCH require auth. **Tentative answer: cite
   DELETE+GET live evidence, assert PUT/PATCH affected by parity
   argument (same handler-entry parse pattern), recommend a
   single fix that covers all four.**
4. **Optional `23514` check-constraint arm (eval Refinement 3)?**
   Out of scope per backlog. **Tentative answer: do not include;
   mention as one-line "future enhancement" footer if at all.**
5. **Cite our eval/evidence paths in the issue body?** Yes, as
   footer "full validation chain" links — same convention as
   plans 01–04.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two-three sentences: `<commit>` closing #17 added a typed-sentinel switch in `errors.go` covering `ErrNotFound`, `ErrHasChildren`, and `isFKViolation` (`23503`). The switch does not cover `22P02 invalid_text_representation`, so a non-UUID path parameter (e.g. `/datastreams/not-a-uuid`) falls through to the default-500 arm. Per RFC 9110 §15.5.1 this should be 400. |
| **Claim** | One bullet: malformed UUID path parameters on all four verbs (DELETE, GET-by-id, PUT, PATCH) across all UUID-keyed resources currently return HTTP 500 instead of 400. |
| **Static evidence** | `git grep` zero-matches for `uuid.Parse\|22P02\|invalid_text_representation` in `internal/api/`. From [`evidence/issue-017/static-analysis-2026-04-30.md`](../evidence/issue-017/static-analysis-2026-04-30.md), refreshed in §6. |
| **Live evidence** | `curl -X DELETE /<resource>/not-a-uuid → 500` and `curl /<resource>/not-a-uuid → 500` for at least Datastream and System. Captured in §6. |
| **Spec authority** | One paragraph: RFC 9110 §15.5.1 is the binding rule (malformed request syntax → 400). RFC 9110 §15.6.1 supports (500 is for unexpected server conditions, not foreseeable client input shapes). OGC 19-072 / OGC 23-001 do not override. |
| **Recommended fix** | Lead with handler-side `uuid.Parse` (or middleware) returning 400 on parse failure; present repository-side `22P02 → 400` arm in `errors.go` as smaller-diff alternative. Defer to maintainer preference. |
| **Scope guard** | "What NOT to touch": no change to the existing `ErrNotFound`/`ErrHasChildren`/`23503` handling; no `23514` arm (separate enhancement); no PUT/PATCH body validation (separate concern); no rename of `errors.go`. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-05-malformed-uuid-pathparam-400.md`](../upstream-issue-reports/report-05-malformed-uuid-pathparam-400.md)

Same `NN` and `<slug>`
(`05-malformed-uuid-pathparam-400`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
