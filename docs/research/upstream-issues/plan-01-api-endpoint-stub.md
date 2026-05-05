# Research Plan 01 — `/api` endpoint OpenAPI stub

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
item **3**, copied verbatim:

> ### 3. #14 — `/api` endpoint stub
> - **Source:** Issue #14 (closed-as-not-planned on fork, comment 4380617500).
> - **Category:** Defect (P2 — binding spec `SHALL` violation).
> - **Summary:** `GET /api` returns an 88-byte JSON literal with only
>   `openapi` + `info`. No `paths`, no `components`, no `servers`.
>   `internal/api/router.go` `getOpenAPISpec` still has the
>   `// TODO: Implement OpenAPI 3.0 spec generation` comment. Verified
>   unchanged on `upstream/main` HEAD `2dc09f7`. Violates OAS 3.0.3
>   §4.7.1.1 (`paths` REQUIRED) and OGC 19-072
>   `/req/landing-page/api-definition-success` clauses B + C
>   (server self-declares `conf/landing-page`).
> - **Recommendation:** Option 3 (serve upstream OGC OAS bundles with
>   patched `servers`, shift media type to `version=3.1`) → fallback
>   Option 1 (`//go:embed` curated bundle).
> - **Status:** Ready to file against
>   `SomethingCreativeStudios/connected-systems-go` after closure pass.
>   Reference our eval + evidence files so maintainer has full chain.

Tier: **A** (P2 spec violation, standalone, not bundled).

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#14` — closed as not-planned on the fork
  via closure-pass comment **4380617500** (closure rationale: this is a
  cs-go upstream defect, not a fork-only concern).
- No other fork issues bundle into this filing.

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-014.md`](../issue-evaluations/issue-014.md) | Full rigour-first evaluation. Verdict KEEP. Severity assessment, fix-option ranking, spec-authority precision note. |
| [`../evidence/issue-014/static-analysis-2026-04-30.md`](../evidence/issue-014/static-analysis-2026-04-30.md) | `internal/api/router.go:255-267` excerpt with the `// TODO` comment; confirmation that no `//go:embed` and no static OAS file exist anywhere in the repo. |
| [`../evidence/issue-014/live-test-2026-04-30.md`](../evidence/issue-014/live-test-2026-04-30.md) | `curl -i` capture of `GET /api` against the live cs-go-head deployment showing the 88-byte body with only `openapi` + `info`. |
| [`../evidence/issue-014/spec-authority-2026-04-30.md`](../evidence/issue-014/spec-authority-2026-04-30.md) | Verbatim clauses from OAS 3.0.3 §4.7.1.1 and OGC 19-072 `/req/landing-page/api-definition-success` B + C. |

## 4. Maintainer triage signal

**None.** Issue #14 was filed during the closure pass and was not
triaged by the upstream maintainer before the fork-side closure. There
is no maintainer quote to anchor framing to. Implications:

- Tone is **first-time surfacing** to upstream, not pushback or
  follow-up. Lead with empirical evidence and spec authority; don't
  reference any prior conversation.
- We do not know the maintainer's preferred fix shape. Present the
  three options from the eval with our ranking and rationale, but
  defer the decision to them.

## 5. Upstream commits relevant to the area

Re-verification (Step 4.2) must capture the current state with these
commands. As of the last check (2026-04-30):

- `upstream/main` HEAD: `2dc09f7` — verified unchanged at
  `internal/api/router.go:255-267` since the eval was written.
- No commits touch `getOpenAPISpec` between the eval and now.
- Sibling commit context: `2dc09f7` itself shipped unrelated GORM
  serialization fixes (Datastream `json:"-"`); we do **not** cite it
  in the upstream filing — it's only relevant as a HEAD anchor.

The report must re-verify the SHA hasn't advanced past `2dc09f7` before
filing. If it has, re-run the static-analysis check on the new HEAD.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these on the current
`upstream/main` HEAD before writing the report's findings sections, and
must paste the trimmed output into the report's §2 "Re-verification
record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'
git show upstream/main:internal/api/router.go | Select-String -Pattern 'getOpenAPISpec|TODO|openapi.*3\.0\.0' -Context 2,2
git grep -E 'go:embed|openapi.*\.(yaml|json)' upstream/main -- internal/
```

```pwsh
curl.exe -i 'https://129-80-248-53.sslip.io/csapi-go-head/api'
curl.exe -s 'https://129-80-248-53.sslip.io/csapi-go-head/conformance' | python -m json.tool | Select-String -Pattern 'landing-page|oas30'
```

Expected on unchanged state:
- HEAD log line shows `2dc09f7` (or later with no router.go diff).
- `getOpenAPISpec` body still contains the hard-coded 2-field literal
  and `// TODO: Implement OpenAPI 3.0 spec generation`.
- `git grep` returns zero hits for `go:embed` and OAS files.
- Live `curl` returns `HTTP/2 200`, `content-type:
  application/vnd.oai.openapi+json;version=3.0`, ~88-byte body.
- Conformance declares `/conf/landing-page` and does **not** declare
  `/conf/oas30`.

If any of these differ from expected, **stop drafting** and surface the
delta — the issue framing may need adjustment or the defect may be
fixed.

## 7. Spec-authority sources

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OAS 3.0.3 §4.7.1.1** — root OpenAPI Object, `paths` field is REQUIRED | "OpenAPI Specification" under *Supporting Specifications* (`https://spec.openapis.org/oas/latest.html`); we cite the 3.0.3 pin because the server self-asserts 3.0 in its `Content-Type` | Hard schema rule the server's response fails. Primary spec authority. |
| **OGC 19-072** — OGC API – Common Part 1, `/req/landing-page/api-definition-success` clauses **B** and **C** | "OGC API - Common" under *Related OGC Standards* (`https://docs.ogc.org/is/19-072/19-072.html`) | Strict `SHALL` clauses bound to cs-go because its `/conformance` declares `…/ogcapi-common-1/1.0/conf/landing-page`. |
| **OGC 19-072** — `/rec/landing-page/api-definition-oas` (SHOULD-level) | same list entry | Secondary, supports the recommendation but not load-bearing. |

**Precision adjustment to record in the report:** cs-go does **not**
declare `/conf/oas30`, so the stricter `/req/oas30/*` requirements
(`oas-definition-2`, `completeness`, `oas-impl`) are **not** directly
binding. The binding `SHALL` for this filing is
`landing-page/api-definition-success` clauses B + C alone. The OAS
3.0.3 schema rule (`paths` REQUIRED) is binding regardless of OGC
conformance declarations because the server self-asserts OAS 3.0 in
its `Content-Type`.

**Out-of-scope sources (do not cite):**
- OGC 23-001 / 23-002 (CSAPI Part 1 / Part 2) — these are the standards
  the server *implements*, not the standard the `/api` endpoint
  *violates*. Mentioning them dilutes focus.
- camptocamp/ogc-client and CSAPI client-side requirements documents —
  irrelevant to a server-side `/api` defect.

## 8. Open questions

1. **Fix-option presentation.** Lead with Option 3 (OGC OAS bundles)
   alone, or present all three (3 → 1 → 2) with our ranking? The eval
   recommends 3 with 1 as fallback. **Tentative answer: present all
   three with ranking and rationale**, since maintainer triage signal
   is absent and they should choose.
2. **Severity framing.** The issue body labels P2-Important. Eval
   concurs with caveat that P1 would be defensible if formal OGC
   certification is roadmapped. **Tentative answer: keep P2** — we
   don't know cs-go's certification ambitions, and P2 is conservative.
3. **Should we mention the `version=3.1` media-type implication of
   Option 3?** OGC's authoritative bundles are OAS 3.1; adopting them
   means the `Content-Type` header changes from
   `application/vnd.oai.openapi+json;version=3.0` to `version=3.1`.
   **Tentative answer: yes, mention explicitly** — it's a non-obvious
   side-effect maintainer needs to know about up front.
4. **Fork-side workaround note.** Should the upstream issue mention
   anything about how downstream consumers (us) cope today? **Tentative
   answer: no** — fork-internal context belongs in the report's §9
   "Fork-side context" section but not in the public-facing extract.
5. **Cite our eval/evidence paths in the upstream issue body?** They
   live in this fork repo (`OS4CSAPI/connected-systems-go`) which is
   public, so the maintainer can read them. **Tentative answer: yes,
   include as "full validation chain" footer links** — cheap,
   consistent with how maintainer responded well to evidence-rich
   filings during the closure pass.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | One paragraph: cs-go advertises `/api` as `service-desc` on the landing page; this filing reports that the response body fails OAS 3.0 schema and one OGC `SHALL` clause. |
| **Claim** | Two-bullet list: (a) `paths` REQUIRED missing from response; (b) response not consistent with declared `Content-Type` and `/conf/landing-page` API-definition contract. |
| **Static evidence** | Excerpt of `internal/api/router.go:255-267` showing the hard-coded 2-field literal and the `// TODO` comment. From [`evidence/issue-014/static-analysis-2026-04-30.md`](../evidence/issue-014/static-analysis-2026-04-30.md). |
| **Live evidence** | `curl -i` request/response capture against `https://129-80-248-53.sslip.io/csapi-go-head/api`. From [`evidence/issue-014/live-test-2026-04-30.md`](../evidence/issue-014/live-test-2026-04-30.md), refreshed during Step 4.2. |
| **Spec authority** | Verbatim clauses for OAS 3.0.3 §4.7.1.1 + OGC 19-072 `/req/landing-page/api-definition-success` B and C. With precision note that cs-go doesn't declare `/conf/oas30`, so the binding SHALL is from `landing-page` alone. |
| **Recommended fix** | Three options (3 → 1 → 2) with one-line rationale each. Highlight Option 3's `version=3.1` implication. |
| **Scope guard** | "What NOT to touch": the landing page itself (it's correct), the `/conformance` endpoint (correct), the existing JSON encoder, anything outside `internal/api/router.go`'s `getOpenAPISpec` function (or wherever the new bundle is wired in). |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-01-api-endpoint-stub.md`](../upstream-issue-reports/report-01-api-endpoint-stub.md)

Same `NN` and `<slug>` (`01-api-endpoint-stub`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
