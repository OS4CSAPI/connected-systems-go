# Research Plan 03 — ControlStream `Systems` field JSON leak

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
item **5**, copied verbatim:

> ### 5. ControlStream `Systems` field JSON leak
> - **Source:** Issue #15 closure (§3.4).
> - **Category:** Defect (P2 — sibling of fix that landed on Datastream).
> - **Summary:** `2dc09f7` added `json:"-"` to `Datastream.Systems` to prevent
>   the GORM many2many slice from serializing into API responses. The exact
>   same field exists on `ControlStream` and was not given the same tag, so
>   `GET /controlstreams/{id}` still leaks the join data.
> - **Status:** Ready to file after closure pass.

Tier: **A** (defect, standalone, not bundled).

> **⚠️ Backlog accuracy flag.** The backlog labels this **P2 — sibling
> of fix that landed on Datastream**, but the original fix on
> Datastream (issue #15) was triaged and accepted as **P3-Minor**.
> The sibling cannot be more severe than the original — they are the
> same defect on a different struct. The eval at
> [`../issue-evaluations/issue-015.md`](../issue-evaluations/issue-015.md)
> §3.6 explicitly concludes "P3-Minor is exactly right. P2 would imply
> functional or strict-spec impact; neither exists." Severity for this
> filing should be **P3-Minor**, matching the parent. Step 4.4 must
> correct the backlog entry.

## 2. Source fork issue(s)

- `OS4CSAPI/connected-systems-go#15` — closed via upstream commit
  `2dc09f7` (Datastream side fixed).
- This filing is the **§3.4 sibling** that issue #15's own scope rule
  ("If you find more, file separately, one issue per leaking type")
  explicitly defers to a separate filing.
- No other fork issues bundle in. Note: backlog item 6 (six latent
  untagged slices on `System` / `Procedure`) is the broader
  **hardening** sibling identified in eval §3.5; it does not bundle
  here per the same one-issue-per-leaking-type rule and per backlog
  policy ("file only if maintainer wants schema-side hardening").

## 3. Existing evaluation artifacts

| Path | Provides |
|---|---|
| [`../issue-evaluations/issue-015.md`](../issue-evaluations/issue-015.md) | Full evaluation. §3.4 records the ControlStream sibling at `internal/model/domains/control_stream.go:53` with identical untagged `Systems []System` field. §3.6 fixes severity at P3-Minor. §3.7 confirms the one-line fix is exact and minimal. |
| [`../evidence/issue-015/static-analysis-2026-04-30.md`](../evidence/issue-015/static-analysis-2026-04-30.md) | Cited line, surrounding convention sweep, custom-marshaller check. Documents that all five sibling FK fields on `ControlStream` carry `json:"-"` and that no `MarshalJSON` exists on `ControlStream`, so `json:"-"` is sufficient. |
| [`../evidence/issue-015/spec-authority-2026-04-30.md`](../evidence/issue-015/spec-authority-2026-04-30.md) | Canonical CSAPI Part 2 `controlStream.json` / `baseStream.json` schemas — no `Systems` property defined; no `additionalProperties: false`. RFC 7493 §4.3 SHOULD-grade interoperability rule. |
| [`../evidence/issue-015/live-test-2026-04-30.md`](../evidence/issue-015/live-test-2026-04-30.md) | Datastream-side reproducer (relevant for shape comparison only). HEAD had zero controlstreams at eval time, so the ControlStream leak could not be demonstrated live; **fresh live reproducer must be captured during Step 4.2**. |

## 4. Maintainer triage signal

**Strongly positive.** The maintainer explicitly accepted the parent
defect (#15), shipped the exact one-line fix in `2dc09f7`, and the
sibling has the identical fix shape. Implications:

- Tone: **direct sibling, same fix, same convention** — short and
  factual. No need to relitigate spec authority or convention; the
  maintainer's own merged commit establishes both.
- Pitch as: "matching the fix you applied in `2dc09f7` on Datastream;
  ControlStream carries the identical field and was not updated in
  the same commit."

## 5. Upstream commits relevant to the area

- **`2dc09f7`** — *the parent fix.* Added `json:"-"` to
  `Datastream.Systems` at `internal/model/domains/datastream.go:57`.
  This is the commit to cite as both precedent and prior art.
- **`d2d1347`** — relevant adjacent commit (removed `SystemLink` from
  Datastream / ControlStream domain models, projected `system@link`
  via formatter). Cite only if Step 4.2 finds it has reshuffled
  `control_stream.go` line numbers.
- Re-verification (Step 4.2) must capture `upstream/main` HEAD SHA at
  filing time and re-locate the field.

## 6. Re-verification commands

The drafter (Step 4.2) must run all of these and paste the trimmed
output into the report's §2 "Re-verification record":

```pwsh
cd C:\Users\sbolling\Documents\connected-systems-go
git fetch upstream
git log -1 upstream/main --format='%H %s'

# Confirm sibling defect still present:
git show upstream/main:internal/model/domains/control_stream.go | Select-String -Pattern 'Systems\s+\[\]System' -Context 2,2

# Confirm parent fix still in place (sanity):
git show upstream/main:internal/model/domains/datastream.go | Select-String -Pattern 'Systems\s+\[\]System' -Context 1,1

# Confirm no custom MarshalJSON on ControlStream:
git show upstream/main:internal/model/domains/control_stream.go | Select-String -Pattern 'MarshalJSON'

# Convention sweep (sibling FK fields on ControlStream should all carry json:"-"):
git show upstream/main:internal/model/domains/control_stream.go | Select-String -Pattern 'json:"-"'

# Recent history on the file (look for any reshuffling since eval):
git log --oneline upstream/main -- internal/model/domains/control_stream.go | Select-Object -First 5
```

```pwsh
# Live re-verification — POST a controlstream first if none exist, then GET:
curl.exe -s 'https://129-80-248-53.sslip.io/csapi-go-head/controlstreams?limit=1' | ConvertFrom-Json | ConvertTo-Json -Depth 6
# Look for top-level "Systems" key (capital-S) in the response object.
```

Expected on unchanged state:
- `control_stream.go` still contains
  `Systems []System `gorm:"many2many:system_controlstreams;"``
  with **no** `json:"-"` tag.
- `datastream.go` shows the parent fix in place
  (`Systems []System `gorm:"many2many:system_datastreams;" json:"-"``).
- No `MarshalJSON` on `ControlStream`.
- All sibling FK fields on `ControlStream` carry `json:"-"`.
- Live `GET /controlstreams` returns at least one object with
  `"Systems": null` at top level.

If the controlstream collection is empty live, the drafter must POST
a controlstream first (auth required) or note that live evidence is
limited to static + transitively from the parent fix. The static
evidence alone is sufficient to file.

If the field already carries `json:"-"` on `upstream/main`, **stop
drafting** — defect already fixed.

## 7. Spec-authority sources

This finding has the same spec posture as parent issue #15: it is
**not a strict schema violation**, but it is an RFC 7493 SHOULD-grade
interoperability defect and an internal-convention defect.

All entries below trace to the authoritative-references list at
`OS4CSAPI/ogc-client-CSAPI_2:phase-8/docs/research/references.md`.

| Source | List entry it traces to | Used for |
|---|---|---|
| **OGC 23-002** — CSAPI Part 2 `controlStream.json` / `baseStream.json` (Part 2 OpenAPI bundle) | "OGC API - Connected Systems - Part 2: Dynamic Data" and "OGC API - Connected Systems - Part 2: OpenAPI Specification" under *OGC CSAPI Standards* | Establishes canonical ControlStream property set. None named `Systems` (any case). Schemas do not declare `additionalProperties: false`, so the leak is permissive-not-prohibited. |
| **RFC 7493** §4.3 (I-JSON) | "RFC 7493 — The I-JSON Message Format" under *IETF RFCs* | The interoperability SHOULD: "senders that wish to be widely interoperable SHOULD only emit … members that are defined in the protocol." Provides the soft-binding rule that converts this from "stylistic" to "violates a documented interoperability principle." |
| **JSON Schema 2020-12** `additionalProperties` default semantics | "JSON Schema Validation 2020-12" under *JSON / OAS* | Backstop for "permissive-not-prohibited" framing; explains why this is a SHOULD not a SHALL. |

**Out-of-scope sources (do not cite):**
- OAS 3.0.3 — not the binding standard for response-body content rules here.
- OGC 23-001 (Part 1) — defines neither Datastream nor ControlStream.
- OGC 19-072 (OGC API – Common) — not relevant; this is a domain-resource
  serialization issue, not a landing-page / API-definition issue.

## 8. Open questions

1. **Severity declaration.** Backlog says P2; eval and parent issue
   say P3-Minor. **Tentative answer: file as P3-Minor**, matching the
   parent and the eval's explicit assessment.
2. **Cite the parent commit `2dc09f7` as the precedent?** Strongly
   yes — it's the maintainer's own merged fix to the identical
   defect shape. It justifies both the spec posture and the fix
   shape without reasoning from first principles. **Tentative answer:
   yes, lead the issue with this.**
3. **Bundle the §3.5 latent-slices hardening into this filing?** No —
   eval explicitly recommends a separate hardening filing, and
   backlog item 6 is the dedicated tracker for it (conditional on
   maintainer interest). Keep this filing narrow. **Tentative
   answer: no, mention as a one-line "out-of-scope sibling" note
   only.**
4. **Live evidence required, or is static-only sufficient?**
   Static is unambiguous; live may be hard to capture if no
   controlstreams exist on the live instance and POSTing requires
   privileged auth. **Tentative answer: static evidence is
   sufficient to file; capture live opportunistically during Step
   4.2 if a controlstream is available, otherwise note absence with
   the explanation that the static evidence is unambiguous and the
   parent-commit precedent confirms the fix shape.**
5. **Cite our eval/evidence paths in the issue body?** Yes, as
   footer "full validation chain" links — same convention as
   plans 01 and 02.

These tentative answers are pre-decisions for the report; if user
disagrees during the report-review step, they get revisited.

## 9. Drafting checklist

For when the report's "Public-facing extract" is being assembled:

| Issue-template section | Content source |
|---|---|
| **Context** | Two sentences: `2dc09f7` fixed the `Systems` field on `Datastream` per issue #15. The identical untagged field exists on `ControlStream` and was not updated in the same commit; per #15's "one issue per leaking type" rule, this is the dedicated sibling filing. |
| **Claim** | One bullet: `internal/model/domains/control_stream.go` `ControlStream` carries `Systems []System `gorm:"many2many:system_controlstreams;"`` with no `json:"-"` tag, causing `encoding/json` to emit `"Systems": null` (capital-S, top-level) in every controlstream HTTP response. |
| **Static evidence** | Excerpt of the offending struct field; one-line confirmation that all sibling FK fields on `ControlStream` carry `json:"-"`; one-line confirmation that no `MarshalJSON` exists on `ControlStream`. From [`evidence/issue-015/static-analysis-2026-04-30.md`](../evidence/issue-015/static-analysis-2026-04-30.md). |
| **Live evidence** | If a controlstream exists at filing time: `curl … /controlstreams?limit=1 \| jq` showing `"Systems": null` at top level. If not: cite parent issue #15's live evidence on Datastream as transitive precedent (same handler shape, same code path). |
| **Spec authority** | One paragraph: not a strict schema violation (CSAPI Part 2 `controlStream.json` does not declare `additionalProperties: false`). Is an RFC 7493 §4.3 SHOULD interoperability defect and an internal-convention defect. Same spec posture as parent #15. |
| **Recommended fix** | One-line diff: append `json:"-"` to the existing struct tag, exactly mirroring `2dc09f7`. |
| **Scope guard** | "What NOT to touch": no rename; no lower-cased `systems` field (not in canonical schema); no sweep of the six latent slices on `System` / `Procedure` (separate hardening filing — backlog item 6); no migration / GORM tag changes. |

## 10. Report file path

After this plan is reviewed and approved, the synthesis report goes at:

[`../upstream-issue-reports/report-03-controlstream-systems-json-leak.md`](../upstream-issue-reports/report-03-controlstream-systems-json-leak.md)

Same `NN` and `<slug>`
(`03-controlstream-systems-json-leak`) as this plan.

---

## 11. Filing record

_(populated after Step 4.3 — leave empty until issue is filed)_

- Upstream issue URL: _pending_
- Filed: _pending_
</content>
