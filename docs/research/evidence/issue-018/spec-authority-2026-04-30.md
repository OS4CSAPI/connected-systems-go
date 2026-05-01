# Issue #18 — Spec authority

Date: 2026-04-30.

## OGC API — Connected Systems, Part 1 (24-001)

The Datastream resource schema defines `phenomenonTime` and `resultTime` as
time intervals. The published JSON encoding section ("Encoding for JSON" /
"timeInterval" — see SWE Common Data Model and CSAPI Part-1 §7.1.4) requires
each element of the interval to be either:

- an ISO 8601 / RFC 3339 instant string, or
- the literal `"now"`, or
- the literal `".."` (open end).

Numeric epoch seconds are **not** a valid encoding for these fields. JSON
arrays containing JSON numbers in those positions are not conformant with the
schema.

CSAPI Part 1 does not normatively prescribe the HTTP error response for
ill-formed time intervals — but it does require (§5.1, "General requirements
for resources") that a server MUST validate received resources against the
declared schema and MUST reject non-conformant requests. The standard pattern
elsewhere in the specification family (e.g. OGC API — Features Part 4
"Create, Replace, Update, Delete") is `400 Bad Request` with a problem
details body for malformed input.

## RFC 7493 (I-JSON) §3.4 — type strictness

The CSAPI schema is published as JSON Schema with `format: date-time` for
interval elements. Per RFC 7493 §3.4, when a schema declares a value as a
string with `format: date-time`, a JSON number in that position is a type
error and must not be silently coerced or dropped.

## Conclusion

The current behavior — accepting `[1773100000.0, null]` with **201 Created**
and silently storing NULL columns — is non-conformant on two distinct points:

1. It accepts an input shape (JSON number) that the schema forbids.
2. It silently drops the field rather than rejecting the request, producing a
   record that does not represent the client's intent.

A spec-conformant server should respond with `400 Bad Request` (or `422`)
with a problem-details body identifying the offending field. This is the
remedy proposed in the issue body and supported by RFC 7493 §3.4 and the
CSAPI Part-1 general resource-validation requirement.

The data-integrity severity (P2) does not depend on the spec analysis: even
if the spec were silent, silently materializing a record whose stored
representation is not what the client sent is a data-integrity defect on its
own merits.
