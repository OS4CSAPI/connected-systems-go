# Issue #24 — Spec authority

Date: 2026-04-30.

## Body's spec citation — independently verified

Body cites `.tmp-csapi-part2.yaml` lines 318-322 for the inline `@link`
schema with `format: uri`. Local copy:
`docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`.

### 1. Schema definition for inline `system@link` (lines 312-324)

```yaml
- properties:
    system@link:
      description: Link to the system producing the observations
      readOnly: true
      $schema: https://json-schema.org/draft/2020-12/schema
      title: Link
      type: object
      required: &ref_11
        - href
      properties: &ref_12
        href:
          description: URL of target resource
          type: string
          format: uri
```

The same `*ref_11` / `*ref_12` references are reused for ControlStream
`system@link` (line 3124) and (by similar reuse) the rest of the inline
link properties enumerated in the body. So **all** inline `@link`
properties resolve to the same `Link` schema with `href: format: uri`.

### 2. Worked examples in the spec — uniformly absolute

Lines 1922-1929 (Datastream example):

```yaml
system@link:
  href: https://data.example.org/api/systems/123
  uid: urn:x-ogc:systems:001
featureOfInterest@link:
  href: https://data.example.org/api/collections/buildings/items/754
samplingFeature@link:
  href: https://data.example.org/api/samplingFeatures/4478
```

Lines 3547-3550 (Command example):

```yaml
system@link:
  href: https://data.example.org/api/systems/4722256
  uid: urn:x-ogc:systems:CAM001
```

**Every** inline `@link.href` in the spec's worked examples is an
absolute `https://...` URI. Body's claim that the absolute form
"matches the spec example" is exact.

### 3. Distinguishing schema — `format: uri-reference` (line 549-550)

For comparison, the spec uses `format: uri-reference` (which permits
relative) only in an XLink-style nested association role schema (lines
547-557, with sibling `role`/`arcrole`/`title` properties — XLink
semantics). The OGC Link object used by the inline `@link` properties
explicitly uses `format: uri`, not `format: uri-reference`.

## RFC 3986 + JSON Schema 2020-12 nuance — important refinement

`format: uri` is an **annotation** in JSON Schema 2020-12 by default —
it does not strictly *fail* validation unless the validator opts into
the format-assertion vocabulary (`https://json-schema.org/draft/2020-12/vocab/format-assertion`).
OpenAPI 3.1 inherits this default-annotation behaviour.

Practical implications:

- A **lax** OAS31 validator will accept the relative form silently.
- A **strict** OAS31 validator (or one configured with
  `format-assertion`) will flag the relative form as a `format` error.
- RFC 3986 §3 distinguishes "URI" (absolute, has scheme) from
  "URI-reference" (URI or relative reference). `format: uri` per
  RFC 3986 §3 means absolute.

So the body's claim "strict OAS31 validators may flag the relative
form" is technically correct. The refinement: relative form is a
**format-vocabulary lint**, not a hard schema violation. Severity
P3 (enhancement / minor conformance gap) is the right framing — it is
not a hard interoperability break.

## Internal-consistency case (independent of strict validation)

Even ignoring strict-validator behaviour, the asymmetry within a single
response (inline `@link` relative; `links[]` array absolute) is a real
ergonomic concern:

- A client that consumes only `system@link.href` must base-resolve
  against the request URI.
- A client that consumes only the `links[]` entry with
  `rel: ogc-rel:systems` does not need to.
- Both forms refer to the same underlying resource — having two
  different encodings in the same response is internally inconsistent.

## Conformance verdict

- **Not a hard non-conformance** — relative form is parseable by
  RFC 3986 §5 base-resolution rules.
- **Lint-level non-conformance** for strict OAS31 validators using
  format-assertion vocabulary.
- **Internal-consistency violation** vs. the same server's `links[]`
  array convention.
- **Spec-example mismatch** — every spec example uses absolute form.

P3 enhancement is the correct severity.

## Recommendation aligned with body

1. Apply `formaters.ToFunctionalAssociationHref(...)` at the JSON
   formatter sites (per body) — minimum-intrusion fix, makes responses
   correct.
2. Additionally apply at the handler-synthesis sites
   (`datastream_handler.go:136`, `control_stream_handler.go:145`) so
   storage is also normalised — defence in depth.
3. The helper is idempotent on absolute values, so applying it
   broadly is safe.
4. Acceptance criterion: every inline `@link.href` in JSON responses
   should match the regex `^https?://`.
