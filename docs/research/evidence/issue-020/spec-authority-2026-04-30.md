# Issue #20 — Spec authority

Date: 2026-04-30.

## OGC API — Connected Systems, Part 1

The Observation `resultTime` field is declared in the schema as a string
(ISO 8601 / RFC 3339 instant). The conformance question — "may a client
send a JSON number for this field?" — is resolved by the spec: no, the
field MUST be a string. This issue (#20) is *not* about that conformance
question (which is the data-integrity story in #19). It is about the
*quality* of the error response when a client sends ill-formed input.

CSAPI Part 1 does not normatively prescribe error-response wording, but
§5.1 ("General requirements for resources") requires the server to reject
non-conformant requests, and the OGC API family generally references RFC
7807 (Problem Details) as the recommended error-response shape.

## RFC 7807 — Problem Details for HTTP APIs

RFC 7807 §1: "[...] this specification defines simple JSON [...] document
formats to suit this purpose. They are designed to be reused by HTTP APIs,
which can identify distinct 'problem types' specific to their needs."

RFC 7807 §3 specifies that a problem details object should include a
human-readable `title` and `detail` such that "the API consumer can use
[the response] to identify the problem". Conflating ≥5 distinct client
error classes into a single message defeats that requirement — the
consumer cannot identify which problem occurred.

The current cs-go error response (`{"error":"resultTime is required"}`) is
already a deviation from RFC 7807 (no `type`, `title`, `status`, `detail`,
`instance` fields), but that's a separate concern. The minimum acceptable
remediation for #20 is to make the existing free-form `error` string
honestly distinguish the actual cause.

## RFC 8259 §3 — JSON value model

RFC 8259 explicitly distinguishes `null` from missing-key-in-object:
"`null` represents the absence of a value, but it is itself a value of
type 'null', distinct from a member that is missing." A client sending
`{"resultTime": null}` is making a different statement than a client
omitting the key entirely. The current implementation collapses both to
the same message; the proposed fix's choice to also collapse them
(treating `null` as equivalent to "missing") is defensible because the
practical client correction is the same in both cases. This is not a
spec-conformance issue.

## Conclusion

P3 (UX) classification is appropriate. No spec violation in the strict
sense (CSAPI doesn't prescribe error-message wording), but:

1. The current single message defeats the spirit of RFC 7807 §3
   ("identify the problem").
2. The misleading message wastes developer time during integration —
   precisely the cost the issue body cites.
3. The proposed fix shape is straightforward and low-risk (changes only
   the error-message branching, not the success path).

The remediation is independent of the data-integrity defect in #19 — but
they're naturally bundled because both touch the same decoder block.
Applying #19's fix shape (explicit type-check before parse) automatically
provides the disentangled error messages that #20 requires.
