# Issue #19 — Spec authority

Date: 2026-04-30.

## OGC API — Connected Systems Part 1, §11 (Observation resource)

The `Observation` resource schema defines `phenomenonTime` and `resultTime`
as ISO 8601 instants (RFC 3339-compatible). The published JSON encoding
requires each field to be a string in instant form. JSON numbers are not a
valid encoding.

CSAPI Part 1 §5.1 ("General requirements for resources") requires the server
to validate received resources against the declared schema and reject
non-conformant requests. The standard pattern in the OGC API family for
ill-formed input is `400 Bad Request` with a problem-details body.

## RFC 7493 (I-JSON) §3.4 — type strictness

A JSON Schema field declared as `type: string, format: date-time` must not
silently accept a JSON number. Clients are entitled to expect that wrong-type
input is rejected, not silently dropped.

## Asymmetry-as-defect

Even setting spec analysis aside, internal consistency within a single API
surface is a baseline reasonable-behaviour expectation (cf. issue #17's
DELETE-handler asymmetry). When semantically-equivalent fields on the same
resource exhibit opposite behaviour for identical wrong-type input, clients
cannot derive one's behaviour from the other — which is the worst possible
state (silent data loss for one, error for the other, with the error message
misleadingly suggesting the field was missing rather than wrong-typed).

## Conclusion

The current behaviour — accepting `phenomenonTime: 1773100000` with **201
Created** and silently substituting `resultTime` as the stored value — is
non-conformant on three points:

1. Accepts an input shape (JSON number) the schema forbids.
2. Silently materialises a record that does not represent the client's
   intent.
3. Inconsistent with the sibling `resultTime` rejection (asymmetry within
   one resource decoder).

A spec-conformant server should respond with `400 Bad Request` carrying a
specific message such as `"phenomenonTime must be an RFC 3339 string"`. The
remedy proposed in the issue body is exactly this shape and is correct.

P2 (data-integrity) classification is appropriate: silent acceptance with
substitution is worse than silent drop because the GET response looks
plausibly correct, defeating client-side validation.
