# OGC 23-002 (CSAPI Part 2) — Observation time-field schema

Source: `.tmp-csapi-part2.yaml` (bundled OAS31 from `OGC/ogcapi-connectedsystems`), bundled excerpt around line 2618:

```yaml
phenomenonTime:
  description: Time at which the observation result is a valid estimate of the sampling feature property(ies). Defaults to the same value as `resultTime`.
  $schema: https://json-schema.org/draft/2020-12/schema
  title: Time Instant
  type: string
  format: date-time
resultTime:
  description: Time at which the observation result was generated.
  $schema: https://json-schema.org/draft/2020-12/schema
  title: Time Instant
  type: string
  format: date-time
required:
  - id
  - datastream@id
  - resultTime
```

**Schema verdict.** JSON Schema `type: string` + `format: date-time` (RFC 3339 §5.6 per JSON Schema validation spec) means **only ISO 8601 / RFC 3339 strings are valid encodings**. Numeric epoch is **not spec-compliant**.

**Implication for issue #3.** cs-go's rejection of numeric `resultTime` is conformant with the published encoding rules. SensorHub's silent acceptance of numeric epoch is a **server-side leniency / non-conformant convenience extension**. The principal defect in the issue body is therefore not the rejection itself, but (a) the misleading rejection message, (b) the silent acceptance of numeric `phenomenonTime`, and (c) the silent acceptance of numeric `TimeRange` arrays in Datastream creation (see static-analysis evidence).
