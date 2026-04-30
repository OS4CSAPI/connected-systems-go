# Issue #13 — Evaluation

**Issue:** [#13](https://github.com/OS4CSAPI/connected-systems-go/issues/13) — `POST /systems/{id}/datastreams silently drops stray top-level uid field`
**Reporter framing:** P2-Important, category API Design / Data Integrity. Drafted in `docs/research/proposed-issues/stray-uid-silently-dropped.md` during the evaluation of issue #1, then promoted to a real issue.
**Repo @:** `c6ca4c6` (`origin/main`).
**Endpoint:** `https://129-80-248-53.sslip.io/csapi-go-head` (Date `Thu, 30 Apr 2026`).

## Verdict

**Not a defect.** The empirical premise is correct (top-level `uid` is silently dropped on POST), but the framing as a P2-Important data-integrity defect does not survive rigorous re-review. Three independent authorities (CSAPI canonical schema, JSON Schema / OpenAPI defaults, RFC 7493 I-JSON convention) all converge on the same posture: silently ignoring unknown JSON properties is permitted, recommended, or default behaviour. Go's `encoding/json` default is exactly the recommended behaviour. The "data loss" framing imports an assumption that the server should preserve client-sent fields the schema does not define — and that assumption is not grounded in the spec, in convention, or in mainstream API practice.

I drafted this issue myself during the #1 evaluation. Re-evaluating it under rigor-first review identifies it as a self-correction case: the framing was over-strong on first pass and should be withdrawn. Recommendation: close as **wontfix / wrong-framing**.

## Evidence

- **Static:** [evidence/issue-013/static-analysis-2026-04-30.md](../evidence/issue-013/static-analysis-2026-04-30.md). End-to-end deserialize path: `handler → MultiFormatFormatterCollection.Deserialize → FormatterAdapter.Deserialize → DatastreamJSONFormatter.Deserialize → json.NewDecoder(reader).Decode(&datastream)`. No `DisallowUnknownFields` anywhere in the package. `Datastream` struct has no top-level `Properties`, no `json.RawMessage` member, no `map[string]interface{}` catch-all. Silent drop is the standard Go default applied uniformly to every unknown top-level key.
- **Live:** [evidence/issue-013/live-test-2026-04-30.md](../evidence/issue-013/live-test-2026-04-30.md). POSTed a body with three classes of stray fields (`uid`, `foo`, nested `properties`) under `sysA = 666ed3fe-...`. Server returned 201 with no warning headers; GET on the new id (`50f019d6-...`) returned only the schema-defined fields. Behaviour is generic across all three stray classes — not `uid`-specific.
- **Spec:** [evidence/issue-013/spec-authority-2026-04-30.md](../evidence/issue-013/spec-authority-2026-04-30.md). Canonical CSAPI Part 2 `baseStream.json` and `dataStream.json` declare neither `additionalProperties: false` nor `unevaluatedProperties: false`. Per JSON Schema 2020-12 default semantics, unknown properties are permitted by the schema. RFC 7493 §4.3 recommends ignoring unrecognised members. RFC 7231 §6.3.2 places no echo/persist requirement on HTTP 201 Created.

## Why the original framing should be withdrawn

The proposed-issue draft (`docs/research/proposed-issues/stray-uid-silently-dropped.md`) was authored during #1 evaluation, and its core claim — *"silent data-loss surface"* — assumed two things that don't survive scrutiny:

1. **That a server's silent acceptance of an unknown field is a data-loss event.** This frames the client's expectation (that `uid` would persist) as authoritative. But the spec doesn't define `uid` on Datastream; the client's expectation was incorrect to begin with. There is no schema-level "data" to lose, only a client-server mismatch on what the wire shape includes. The mismatch belongs in the client's migration plan, not as a server-side defect.

2. **That `201 Created` carries an implicit echo-back contract.** RFC 7231 §6.3.2 is silent on this. No HTTP convention says "the server must reject or echo every field of the request body." Most JSON APIs ignore unknowns by default; this is what JSON Schema, OpenAPI, RFC 7493, and Go's standard library all converge on.

Once both assumptions are removed, the issue dissolves into a UX preference question: would it be friendlier to clients if cs-go warned on unknown fields? Yes — but that's not a defect, it's a deployer-side decision the upstream maintainer is free to make or decline.

## Why the three proposed options are all flawed

The issue body proposes three remediation options, framed as "the maintainer chooses one of these". Reviewing each against the spec + convention surfaced earlier:

1. **`DisallowUnknownFields` globally.** Breaking change for any client that legitimately uses extension fields (which the spec permits). Goes against I-JSON's RECOMMENDED-ignore posture. Would surface as HTTP 400 on every client that has ever used a forward-compat or vendor-namespaced field. Not spec-mandated, not convention-aligned, and overly disruptive.
2. **Reject only `uid` specifically.** Singles out one key with no spec basis. Brittle; creates asymmetric semantics with no principled rule. The very generality of the silent-drop (uniform across `uid`, `foo`, `properties`) — confirmed live — argues directly against this option.
3. **Re-add no-op `uid` field with `gorm:"-"`.** Resurrects a vestigial field whose only purpose is round-tripping a wire shape the spec doesn't define. Locks cs-go's wire shape to a SensorHub-style convention that isn't part of the canonical schema. Technical debt.

All three are *permissible* (servers may add stricter or more verbose behaviour), but none is *spec-mandated* or *convention-aligned*. The maintainer's actual decision space is "leave the JSON-default alone" vs. "add a 200-line UX feature with tradeoffs."

## What's actually load-bearing

The real concern surfaced in #1 — and which #13 correctly identified as a wire-shape mismatch between the production deployment and HEAD — is **client-side migration coordination**. The OSHConnect-Python publisher fleet was sending `uid` against a server that no longer carries it. The fix for that mismatch is in the client (drop `uid` from POST bodies, or migrate to a server-controlled identity scheme), not in the server (add JSON strictness or echo back vestigial fields).

This client-migration concern is real and is tracked in the OSHConnect-Python integration work. It is **not** a cs-go defect.

## Adjacent finding (out of scope, recommend separate issues)

Cleanup attempt on the test datastream `50f019d6-765b-4f51-94b6-037eae06a235` returned HTTP 500 (with and without `?cascade=true`). This is the same `DELETE /datastreams/{id}` defect already flagged as finding B in #12's evaluation. No new information here; cross-reference to #12 evaluation.

The orphan datastream remains in the HEAD database alongside the three orphans from #12 (`4300e090`, `95146016`, `26a80a70`), pending a maintainer-side wipe.

## Severity

Recommend: **close as wontfix / wrong-framing**. The empirical observation should be preserved (it's a real wire-shape signal), but the P2-Important defect framing should be withdrawn.

Comment posted: [#issuecomment-4356920552](https://github.com/OS4CSAPI/connected-systems-go/issues/13#issuecomment-4356920552).

## Recommendations to issue thread

- Acknowledge the empirical observation reproduces.
- State that on review, the silent-drop behaviour is consistent with the canonical CSAPI Part 2 schema (no `additionalProperties: false`), with JSON Schema / OpenAPI defaults, with RFC 7493 §4.3 ignore-recommended, and with Go's `encoding/json` default.
- Note that all three proposed remediation options are deployer-UX choices rather than defect fixes, and each has tradeoffs that argue against unilateral application.
- Identify the underlying real concern (client-side migration coordination from a pre-`1562201` wire shape) and locate it in the OSHConnect-Python publisher tracker, not as a cs-go server defect.
- Recommend close as wontfix / wrong-framing, with the proposed-issue draft retained for archival but flagged as superseded.

## Methodology notes

1. **Self-issued issues require the same rigor as third-party issues.** Drafting an issue is a low-effort act; rigorous re-evaluation is the gate against false-positive findings. #13 is the second issue we drafted that re-evaluation downgraded (after we walked back parts of earlier proposed-issue drafts during #1's evaluation). Standing rule: every issue we file should be re-evaluated under rigor-first review before any action is taken, even — especially — issues we authored.
2. **Convention-as-baseline framing.** This is a sibling pattern to the SensorHub-as-baseline framing identified in #5/#6/#7/#9. There, an outside server's behaviour was being read into the spec; here, an outside server's behaviour (round-trip `uid`) was being read as the implicit wire-shape contract that cs-go was "violating". The fix is the same: anchor on the canonical spec authority and the relevant RFCs, not on what a comparator implementation does.
3. **Generic vs. specific behaviour matters.** The issue framed `uid` as the specific concern, but live testing showed `foo` and `properties` are dropped identically. When a "specific" defect turns out to be a generic default behaviour, the framing collapses — there is no `uid`-specific code path to fix; there is only the JSON decoder default, which is itself spec-aligned. Always test a control alongside the suspected case to see whether the alleged defect is specific or generic.
4. **`additionalProperties` is the load-bearing schema knob.** When evaluating "is silent acceptance of field X a defect?", the first question is "does the schema declare `additionalProperties: false`?" If not, the schema permits the field; the server's choice to accept-and-ignore is conformant. If yes, accepting it would be a schema violation. CSAPI Part 2 declines to set this knob, which is itself an authorial choice in the direction of permissiveness.
5. **RFC anchoring is cheap and decisive.** Once the scope of the question is "what should an HTTP/JSON API do with unknown fields?", citing RFC 7493 §4.3 and JSON Schema 2020-12 defaults closes the question in seconds. Costless to do; large clarifying value for any issue framed around wire-shape behaviour.
