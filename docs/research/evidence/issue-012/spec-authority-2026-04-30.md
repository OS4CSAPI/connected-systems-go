# Issue #12 — Spec authority (2026-04-30)

Source: `opengeospatial/ogcapi-connected-systems@master`, fetched 2026-04-30.

## Datastream JSON encoding (CSAPI Part 2)

**`api/part2/openapi/schemas/json/baseStream.json`** — properties:

| property | required | notes |
|---|---|---|
| `id` | yes | "Local resource ID. If set on creation, the server may ignore it." `readOnly: true`. |
| `name` | yes | Human readable name. |
| `description` | no | Human readable description. |
| `validTime` | no | TimePeriod. |
| `formats` | yes | Array of strings. |

**No `uid` or `uniqueIdentifier`.**

**`api/part2/openapi/schemas/json/dataStream.json`** = `allOf [baseStream.json, {extension}]`. Extension properties:

`system@link`, `outputName`, `procedure@link`, `deployment@link`, `featureOfInterest@link`, `samplingFeature@link`, `observedProperties`, `phenomenonTime`, `phenomenonTimeInterval`, `resultTime`, `resultTimeInterval`, `type` (enum: status / observation), `resultType` (enum: measure / vector / record / coverage / complex), `live`, `schema`, `links`.

Required at the dataStream level: `name`, `system@link`, `observedProperties`, `phenomenonTime`, `resultTime`, `resultType`, `live`.

**No `uid` or `uniqueIdentifier`.**

## What this means for issue #12's framing

The issue treats `uid` as a Datastream-level property and asks whether its uniqueness should be global, per-system, or dropped. **The canonical CSAPI Part 2 Datastream JSON schema has no `uid` property at all.** Datastreams are identified solely by their server-assigned `id`, and they are intrinsically children of a system through `system@link`.

`uid` is a **System-level** concept (CSAPI Part 1 `system.json` carries it via the corresponding base type). The issue's mental model — "POST a datastream with a URN-style UID and have the server treat that URN as the datastream's persistent identity" — does not appear in the spec.

This means:

- **The spec is silent on Datastream UID uniqueness because the spec has no Datastream UID.**
- The issue's Option A (composite uniqueIndex on `(system_id, unique_identifier)`) imports a System-level concept onto Datastreams; that would be a deviation from the spec, not an alignment.
- The issue's Option B (drop the constraint entirely) is closer to spec-faithful, because it leaves Datastreams identified only by `id` — which matches the schema. **cs-go HEAD already implements this** (commit `1562201` removed the `CommonSSN` embedding from `Datastream`).

## Comparator claim ("SensorHub scopes datastream uniqueness per parent system")

I do not have access to a canonical SensorHub OAS at a known URL to verify or refute this claim, and the issue does not cite one. Whether SensorHub does or does not enforce per-system Datastream UID uniqueness is **not load-bearing** for the cs-go evaluation: even if it does, that is a SensorHub deployment choice that goes beyond what the canonical Datastream schema defines.

This matches the pattern noted in earlier evaluations (#5, #6, #7, #9): SensorHub-as-baseline framing imports SensorHub deployment behaviour as the implicit spec. In this case the comparator is plausibly correct about SensorHub, but the spec it is being read into doesn't define the property under discussion, so the comparison can't yield a spec-compliance verdict against cs-go either way.

## OGC 23-001 §8.3 cited in the issue

The issue cites OGC 23-001 §8.3 ("datastreams are children of systems") to ground the per-system-scope argument. That citation is accurate but does not establish that Datastreams have a UID. Children of a system can be identified within the parent's scope by their server-assigned `id`; that is in fact how the canonical schema models them.

## Conclusion

The spec **does not assign Datastreams a `uid` property**, so:

- There is no spec basis for "global UNIQUE on `datastreams.unique_identifier`" being correct (which the issue argues).
- There is no spec basis for "per-system composite uniqueIndex on `(system_id, unique_identifier)`" being correct (the issue's Option A).
- There **is** a spec-faithful position: Datastream is identified by server-assigned `id`, no UID at all. cs-go HEAD already matches this position.

The issue's framing — "should be scoped per parent system" — is built on a property the canonical Datastream JSON schema does not define. Once that observation is registered, the question "global vs per-system uniqueness" doesn't have a spec-derived answer, and the empirical answer on HEAD ("neither — there is no constraint, because there is no column") is consistent with the canonical schema.
