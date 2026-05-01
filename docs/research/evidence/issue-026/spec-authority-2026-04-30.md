# Issue #26 — Spec Authority

Source: `docs/research/standards/ogcapi-connectedsystems-1.bundled.oas31.yaml`
(OGC 23-001, Connected Systems Part 1, draft).

## `idList` parameter (lines 5775-5793)

```yaml
idList:
  name: id
  description: |-
    List of resource local IDs or unique IDs (URI).
    Only resources that have one of the provided identifiers are selected.
  in: query
  required: false
  schema:
    $ref: '#/components/schemas/idListSchema'
  explode: false
  examples:
    id:
      summary: Local IDs
      value: RES_ID1,RES_ID2,RES_ID63
    uri:
      summary: Unique IDs
      value: urn:example:resource:001,urn:example:resource:033
```

**Verified vs. body**: exact match. Body cites the parameters file in
the source repo (`opengeospatial/ogcapi-connected-systems/.../parameters/idList.yaml`);
that file is bundled into the OAS31 component above. Name, description,
and both worked examples are byte-identical.

## Where `idList` is referenced

`Select-String '\$ref: .#/components/parameters/idList'` returns 14 path
operations using the parameter — one per list endpoint:
- `/systems` (line 122, 206 — root + filtered variants)
- `/systems/{id}/subsystems` (345)
- `/datastreams` (399, 518)
- `/observations` (574)
- `/controlstreams` (601, 717, 748)
- `/properties` (863)
- (plus deployments / procedures / samplingFeatures / featuresOfInterest —
  same pattern)

In every case the parameter `name: id`. There is no `uid` parameter
defined anywhere in the spec.

## Is `?uid=` defined as an alias by any related spec?

Searched both bundled OAS31 files (Part 1, Part 2). No occurrence of
`name: uid` as a query parameter. Only schema property `uid` exists (on
the resource bodies). So `?uid=` has no spec authority and should not
be accepted as a request-side alias unless the project deliberately
adds a non-spec extension.

This validates body's "Why not also accept `?uid=` as an alias?"
section — the rejection rationale is well-grounded in spec hygiene.

## Sibling parameters with `name:` ≠ Go field name

For completeness, several related parameters use a different query name
than their schema slot:
- `parentIdList` → `name: parent` (line 5800)
- `procedureIdList` → `name: procedure` (line 5810)
- `foiIdList` → `name: foi` (line 5820)
- `obsPropIdList` → `name: observedProperty`
- `systemIdList` → `name: system`

All accept dual local-ID-or-URI value form (same description body).
This is a useful additional point for the README change: the
"value can be a local ID or a URI" semantics is shared across **all**
ID-list query parameters, not just `?id=`.

## Severity calibration

P4 docs is correct. The body's recommended wording is well-aligned
with the spec text and the project's actual implementation.
