# Issue #22 — Spec authority

Date: 2026-04-30.

## What the spec says about `optional`

`docs/research/standards/ogcapi-connectedsystems-2.bundled.oas31.yaml`,
line 521 (in the `DatastreamDataComponent` schema):

```yaml
optional:
  description: Specifies if the data for this component can be omitted in the datastream
  type: boolean
  default: false
```

The spec uses the verb **"omitted"** — i.e. literally absent from the
encoded record. It does not say "omitted or null". This is a narrow,
positive statement.

## What the spec says about `null` for component values

I searched the bundled OAS31 for null-related schema constructs:

```
$ Select-String -Pattern 'nullable' ogcapi-connectedsystems-2.bundled.oas31.yaml
(zero matches)

$ Select-String -Pattern '"null"' ogcapi-connectedsystems-2.bundled.oas31.yaml
(zero matches)
```

Zero occurrences of OpenAPI 3.0's `nullable: true` and zero occurrences
of OpenAPI 3.1 / JSON-Schema-2020-12's `"null"` type or `["number","null"]`
union. Quantity component values are typed `number`, Count is `integer`,
Boolean is `boolean`, Time/Category/Text are `string` — none with a null
union.

Per JSON Schema semantics: a schema `type: number` *rejects* `null`.
Therefore a strict OAS31 validator MUST reject `{"baro_altitude": null}`
regardless of the field's `optional` flag — `null` is not in the value
domain at all.

## SWE Common's canonical "no value" mechanism

Per OGC 21-022 (SWE Common 3.0) and confirmed in #21's evaluation, the
spec-canonical way to encode "this component has no value at this
record" is one of:

1. **Omission** — leave the field key out of the JSON object entirely
   (only valid when `optional: true`).
2. **`nilValues` carrier string** — use a declared
   `nilValues[i].value` (one of `"NaN"`, `"Infinity"`, `"+Infinity"`,
   `"-Infinity"`) so the consumer can map back to the `reason` URI
   (`.../missing`, `.../BelowDetectionRange`, etc.).

JSON `null` is **not** in the SWE Common toolkit. A client wanting to
say "no value here" should either omit the key or use the appropriate
nilValue carrier. Accepting `null` as equivalent to absence — as the
issue body proposes — would be a *non-conformant convenience*: the
implementation would silently absorb a value that no spec schema
permits.

## Conformance verdict on the current behaviour

The current server behaviour (reject `null` with `"must be a number"`)
is **spec-conformant**. This contrasts with #21, where the current
behaviour is *non-conformant* (rejects spec-mandated nilValue strings).
The two issues describe surface-similar symptoms ("validator rejects
something my client sends"), but their spec status is opposite.

## Where the issue's framing is still legitimate

Two aspects survive the spec analysis:

1. **UX asymmetry**: T1 (omitted) and T2 (`null`) feel symmetric to a
   developer ("both mean no value"), but produce different HTTP
   outcomes. This is real cognitive friction even though it's
   spec-justified.

2. **Diagnostic clarity** (adjacent finding from live-test T5): the
   error message `"must be a number"` for a `null` value is misleading
   — `null` is not a wrong-type number, it's a *not-permitted JSON
   token*. A clearer message would be:

   ```
   result.baro_altitude is null; null is not a permitted value (use
   omission for optional fields, or a declared nilValue such as "NaN").
   ```

## Recommendation framing

The issue's proposed fix (silently accept null as absence on optional
fields) is **not recommended** because it teaches clients a
non-canonical encoding and weakens schema enforcement. A better
remediation, which preserves spec conformance while addressing the UX
concern the body raises, is:

- **Improve the error message** for `null` values on typed leaf
  components (point clients to omission and `nilValues` as the canonical
  alternatives).
- **Document the asymmetry** in API user docs so encoders know to use
  omission rather than null.
- **Optionally**: behind a documented configuration flag, accept
  `null` as syntactic sugar for absence on `optional: true` fields. Off
  by default (preserves strict OAS31 conformance), opt-in for
  client-friendly deployments.

This converts a "fix the bug" framing into a "improve diagnostics +
documentation" framing that is both spec-aligned and addresses the real
developer pain point.
