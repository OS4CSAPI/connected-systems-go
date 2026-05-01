# Issue #26 — Evaluation

| Field | Value |
|---|---|
| Issue | OS4CSAPI/connected-systems-go#26 |
| Title | [P4 docs] Document that `?id=` accepts both local IDs and URIs |
| Labels | `documentation` |
| Verified HEAD | `5be1d42` |
| **Verdict** | **KEEP** (with one accuracy correction) |
| Severity | **P4 documentation** — confirmed |

## Summary

Body's spec citation, dual-semantics implementation claim, and
publisher-side UX problem are all verified end-to-end. Live testing
shows `?id=urn:...` and `?id=<uuid>` both filter correctly, while
`?uid=...` is silently ignored (observationally identical to
`?foo=bar`). The README **already** has a one-line bullet that mentions
the dual semantics, but it is terse and easy to miss — discoverability
gap is real, just smaller than the body implies. Recommended scope:
expand README + OpenAPI landing with concrete examples and an
explicit `?uid=` clarifying note.

## Verification

- **Static** — see [static-analysis-2026-04-30.md](./evidence/issue-026/static-analysis-2026-04-30.md). `query_params.go:56` reads only `?id=`. Dual semantics SQL `id IN ? OR unique_identifier IN ?` confirmed in **all 8** UID-bearing repositories (System, Datastream, ControlStream, Deployment, Procedure, Property, Feature, SamplingFeature).
- **Live** — see [live-test-2026-04-30.md](./evidence/issue-026/live-test-2026-04-30.md). On 17-system baseline: `?id=urn:test:issue1:sys:1` → 1 hit, `?id=666ed3fe-...` → 1 hit, `?uid=...` → 17 (unfiltered, ignored), `?foo=bar` → 17 (control). Body's claim that `?uid=` is observationally equivalent to an unknown parameter is **exactly right**.
- **Spec** — see [spec-authority-2026-04-30.md](./evidence/issue-026/spec-authority-2026-04-30.md). OAS31 Part 1 lines 5775-5793 define `idList` with `name: id`, dual-form description, and both examples (`RES_ID1,RES_ID2,...` and `urn:example:resource:001,...`). No `name: uid` query parameter anywhere in the spec. Body's quote is byte-identical.

## Where the body is correct

- Spec citation exact (parameters file content reproduced verbatim).
- Implementation claim exact (8 repositories with dual SQL).
- Live UX symptom exact (silent no-op for `?uid=`).
- Severity P4 (docs only) is correct.
- Rationale for rejecting `?uid=` aliasing on the server side is
  spec-hygiene-sound.
- Suggested wording for README is accurate and aligned with spec text.
- Affected-endpoints list (systems, datastreams, controlstreams,
  deployments, procedures, properties, samplingFeatures) matches
  `git grep` of `idList` references in the spec exactly.

## Where the body needs a small accuracy correction

> "What is missing is **discoverability** — there is no client-facing
> doc that says 'the URI form goes into `?id=`, not `?uid=`'."

**README.md:158-167 already includes:**

```
## Query Parameters

Common query parameters across list endpoints:

- `id` - Filter by resource ID or UID
...
```

So the dual semantics **is** documented in one bullet line. The body's
claim is partly true (no examples, no `?uid=` clarification) but
overstates the gap. The recommended scope should be **"expand the
existing line"** rather than **"add a new section that doesn't exist"**.

## Refinements

1. **Acknowledge existing bullet, expand in place** — the README change
   is an enhancement of the existing `## Query Parameters` section,
   not net-new prose. This keeps git blame meaningful and avoids
   creating two parallel documentation surfaces.

2. **Cover sibling ID-list parameters too** — the spec defines the
   same dual-form semantics for **all** ID-list parameters, not only
   `?id=`. cs-go implements `parent`, `procedure`, `foi`,
   `observedProperty`, `system` etc. with similar parsing. The doc
   note "value can be a local ID or a URI" generalises naturally.
   Worth adding one line in the same section.

3. **OpenAPI landing surface check** — the issue body's second
   acceptance-criterion ("link from OpenAPI doc / landing page or
   `/api` page") needs a concrete target. cs-go's spec landing surface
   is at `app/` (Vue) plus a generated `/api` if present. The PR
   should verify whether the spec viewer is sourced directly from the
   bundled OAS31 (in which case the existing `idList` description is
   already shown to users) or whether there's a separate prose page
   that needs the note. **My read**: the bundled OAS31 description
   already says "List of resource local IDs or unique IDs (URI)" with
   examples — so for users who land on the OpenAPI viewer the
   discoverability is **already good**. The README change is the
   higher-leverage fix.

4. **Sharper acceptance criterion** — the body's third bullet ("cross-link from any contributor docs that previously implied a `?uid=` parameter exists") is a no-op since no such cs-go-side docs exist. Drop it; replace with "regression test that GETting `/systems?id=<urn>` returns exactly the system whose `uid` matches".

5. **Optional one-line server warning** — entirely out of scope for #26
   (which is docs-only) but worth recording for a separate ticket: a
   debug-level log line "received unknown query parameter `uid` —
   spec parameter is `id`" would close the loop for publishers without
   any spec extension. Decide later; not part of this PR.

## Recommended PR scope

For a PR closing #26:

1. **Edit `README.md`** § "Query Parameters" — replace the single
   `id` bullet with the body's expanded prose + 3 worked examples
   (local UUID, URI, mixed comma-separated). Adopt the body's
   wording with minor edits to match house style.

2. **Add one paragraph** noting that the same dual-form value applies
   to other ID-list parameters (`parent`, `procedure`, `foi`,
   `observedProperty`, `system`) — generalises the lesson.

3. **Regression test** in `internal/api/*_handler_test.go` (or
   `internal/repository/system_repository_test.go`) asserting that
   GET `/systems?id=urn:test:...` returns the same system as GET
   `/systems?id=<uuid>` for the same record.

4. (Defer) Server-side `?uid=` warning log — separate ticket.

## Verdict rationale

KEEP. P4 docs enhancement well-grounded. Body's claims verify in code,
spec, and live behaviour. One small accuracy correction (README's
existing bullet) does not invalidate the issue — it's still a real
discoverability gap; the fix is just expanding rather than creating.
