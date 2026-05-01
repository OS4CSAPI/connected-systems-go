# Issue #25 — Spec Authority

Source: `docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`
(OGC 23-002, Connected Systems Part 2, draft).

## Link object schema (lines 312-372)

Inline `system@link` (and via shared anchors all other inline `@link` fields)
defined as:

```yaml
system@link:
  description: Link to the system producing the observations
  ...
  title: Link
  type: object
  required:
    - href
  properties:
    href:
      description: ...
      type: string
      format: uri              # see issue #24
    rel:
      description: Link relation type
      type: string
    type:
      type: string             # IANA media type
    hreflang:
      type: string
    title:
      type: string             # human-readable label
    uid:
      type: string             # target's persistent identifier
    rt:
      type: string             # resource type
    if:
      type: string             # interface description
```

**Verified vs. body**:
- Body asserts `href`, `rel`, `type`, `title`, `uid`, `hreflang`, `rt`, `if`
  with only `href` required → matches spec exactly.
- Cited "lines 318-380" in `.tmp-csapi-part2.yaml` ≈ lines 312-372 here
  (line numbering varies with bundling). Schema content matches.

## Reuse via anchors

Spec defines this Link object once and reuses it via `*ref_11` / `*ref_12`
for `procedure@link`, `deployment@link`, `featureOfInterest@link`,
`samplingFeature@link` (DS lines 376-401), and again on ControlStream around
line 3124+, and on Observation/Command. All inline `@link` properties
inherit the same optional-field surface.

## Are the optional fields purely advisory or required-when-known?

Spec text uses "optional" without further constraint. There is no
"SHOULD populate when known" normative language. Therefore:

- Omitting `rel`/`type`/`title`/`uid`/etc. is **spec-compliant**.
- Adding them is **spec-compliant**.
- The argument for adding is **conventional/UX** not normative.

This calibrates issue severity to P4 (enhancement, not conformance) — body
is correct.

## Worked examples

OAS31 example payloads at lines 1922-1929 (Datastream) and 3547-3550
(Command) show inline `@link` with `href` only (no optional fields). So
even the spec's own examples don't populate the optionals. This means
issue #25 is asking the cs-go server to **exceed** the spec examples'
ergonomics, which is fine but not driven by external pressure.

## Suggested-fields review against spec

Body's suggested mapping:

| Field | Body's suggestion | Spec compatibility |
|---|---|---|
| `href` | absolute URL | matches `format: uri` (#24) |
| `type` | `application/geo+json` (system), `application/sml+json` (procedure) | spec says "type" is media-type — both values are valid IANA media types (sml+json registered with OGC SensorML 2.0 binding). ✅ |
| `title` | `system.Name` / `procedure.Name` | spec doesn't constrain Title content; using the resource's display name is conventional. ✅ |
| `uid` | resource UID (urn) | spec field is for target's persistent identifier — perfect match. ✅ |
| `rel` | `parent` for `system@link`, omit for `procedure@link` | spec doesn't constrain rel value; "parent" is a registered IANA relation. ✅ — but the **OGC convention** for "system that owns this datastream" rel is `ogc-rel:hostedOn` or similar (cs-go is using `ogc-rel:systems` in its `links[]` array). For inline `@link`, omitting `rel` is also reasonable since the property name (`system@link`) already conveys the relation. Either choice is valid. |

Caveat on `rel="parent"`: there is a tension between the IANA "parent"
relation (RFC 8288 plus IANA Link Relations registry) and OGC
relation conventions. Recommendation in eval: use OGC conventions
(`ogc-rel:host` or omit) rather than IANA `parent` to be consistent with the
project's `links[]` array style.

## Bottom line

Spec verifies the body's claims. P4 severity is correct. The proposed
enrichment is purely conventional/UX with full spec support.
