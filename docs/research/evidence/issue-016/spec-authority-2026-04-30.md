# Issue #16 — Spec authority note

Date: 2026-04-30.

## Scope of spec authority for this issue

Issue #16 makes three claims, **none** of which depend on a specific OGC API
Connected Systems normative requirement:

| Claim | Authority needed |
|---|---|
| Bug 1 — `?cascade=true` returns 500 due to missing m2m join cleanup | None (internal correctness — the codebase ships `?cascade=true` as its own mitigation API; that mitigation must work) |
| Bug 2 — `DeploymentRepository.Delete` lacks a cascade parameter | None (internal API parity with peer DELETE handlers) |
| Bug 3 — Natural 1:N children have no FK; non-cascade DELETE silently orphans | None (database integrity / data correctness) |

The issue is correctly framed as a **code-defect / data-integrity** report, not
a spec-compliance report. Therefore no fetch of canonical OGC schemas is
required to validate it.

## OGC API Connected Systems Part 2 — DELETE semantics

For completeness, the OGC API Connected Systems Part 2 draft and the OGC API
Common base do not normatively require any specific cascade-vs-restrict
deletion policy. Server implementations are free to either:

1. Reject deletion of resources with dependents (typically with HTTP 409
   Conflict — common practice but not mandated), or
2. Implement transitive deletion under server discretion (the cs-go choice via
   `?cascade=true`).

cs-go has chosen path (2) and exposed `?cascade=true` as the opt-in. The bug
is that this opt-in is non-functional, not that the choice itself is invalid.

## RFC 7807 / RFC 9457 problem detail considerations

Adjacent to but not part of #16: the current 500 response body
`{"error":"Failed to delete X"}` is not RFC 7807 Problem Details JSON, and the
SQL constraint name (`fk_system_datastreams_datastream`) is logged server-side
rather than leaked to the client (verified by `Failed to delete datastream`
being the entire response body). That is correct and should be preserved when
the proposed `mapDeleteError` helper is implemented — keep error class
mapping (409/404/500) but do not surface SQL identifiers.

## Conclusion

Issue #16 is wholly an internal-correctness issue. No spec violation is
claimed, and validation does not require external authority.
