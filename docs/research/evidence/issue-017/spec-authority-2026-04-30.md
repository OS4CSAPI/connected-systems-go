# Issue #17 — Spec authority note

Date: 2026-04-30.

## Authorities relevant to this issue

### RFC 9110 (HTTP Semantics)

The issue body cites RFC 9110 §15.5.10 (409 Conflict) — verified accurate:

> "The 409 (Conflict) status code indicates that the request could not be
> completed due to a conflict with the current state of the target resource.
> This code is used in situations where the user might be able to resolve the
> conflict and resubmit the request."

A foreign-key constraint violation on DELETE fits this exactly: the resource
has dependents whose presence conflicts with the deletion request, the user
can resolve by either removing dependents first or using `?cascade=true`.

§15.5.5 (404 Not Found) is straightforward.

§15.5.1 (400 Bad Request) covers malformed request payloads.

§15.6.1 (500 Internal Server Error):

> "an unexpected condition that prevented it from fulfilling the request"

A predictable, known-class error like "FK conflict" is *not* "unexpected" by
this definition; using 500 for it is a mild semantic abuse. The issue's
critique is well-founded.

### RFC 9110 §9.3.5 — DELETE idempotency

> "DELETE is idempotent."

204 for a missing resource is *permitted* by RFC 9110 — the second invocation
of DELETE on an already-deleted resource is a valid idempotent operation. So
the 5-handler 404 path and the 7-handler 204 path are *both* RFC-compliant in
isolation. The problem is the **inconsistency within the same API surface** —
clients can't generalize.

### RFC 7807 / RFC 9457 (Problem Details for HTTP APIs)

Neither this issue nor the recommended remediation requires switching to RFC
7807 problem-details `application/problem+json` bodies. The current
`{"error":"…"}` shape is a sufficient improvement target. RFC 7807 adoption
would be a separate, larger refactor.

### OGC API Connected Systems Part 2

The OGC API Connected Systems Part 2 draft does not normatively prescribe
status codes for DELETE error conditions beyond the general OGC API Common
guidance. There is no spec violation involved — this is purely an
internal-API-quality issue.

## Conclusion

The HTTP-semantics authorities cited (RFC 9110) are accurate and support the
issue's central recommendation. The proposed `classifyRepoError` helper
produces RFC 9110-compliant status codes for the four classes (404 / 409 /
400 / 500) and is the correct minimal solution.

The 204-on-missing path discovered during live testing is RFC-permissible per
§9.3.5 idempotency, but the inconsistency with the 5 handlers that 404 should
be unified — either move all to 404 (preferred for actionable diagnosis) or
all to 204 (preferred for strict idempotency). Recommendation: 404, because it
preserves the stronger client signal and the 5 handlers that already do this
have validated the pattern.

No external spec authority is required to validate the issue further.
