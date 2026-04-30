# Issue #11 — Spec authority (2026-04-30)

## Parameters in question

The issue concerns three temporal query parameters used on observation collections (and per OGC 23-001r0 §8.7, also on discovery resources):

- `resultTime` — when the result was generated/recorded (CSAPI Part 2 §13.3)
- `phenomenonTime` — when the observed phenomenon occurred (CSAPI Part 2 §13.3)
- `datetime` — common temporal filter inherited from OGC API – Features Part 1 §7.15.4 (applies to systems / deployments / sampling-features collections per OGC 23-001r0 §8.7)

All three parameters share the same value grammar: either an RFC 3339 instant, an interval `start/end` (where `..` may stand in for an open bound), or — for `resultTime` only on observation collections — the keyword `latest` (CSAPI Part 2 §13.3.2 statement D).

## Required parser behaviour (per spec)

1. **Interval form** `start/end`, including `../end` (open-start) and `start/..` (open-end), MUST be recognised. This is OGC API – Features Part 1 §7.15.4 verbatim and is referenced from CSAPI Part 2 §13.3.

2. **Special keyword `latest`** on `resultTime` MUST select only the most recent observation per datastream (CSAPI Part 2 §13.3.2 statement D).

3. **Invalid values** SHOULD result in an error response (OGC API – Features Part 1 §7.15.4: "If the value of a parameter is invalid… the server SHALL return an HTTP 400"). At minimum the server MUST NOT silently treat invalid input as "no filter".

## Authority chain followed

- Title and §-numbers were taken from the issue's own citations to opengeospatial/ogcapi-connected-systems and opengeospatial/ogcapi-features. These are normative documents in the OGC standards process.
- The `latest` keyword is a CSAPI-specific extension; the interval grammar is inherited unchanged from Features Part 1.
- The current implementation in `time_range.go` already shows partial awareness: the `ToTimeRange(string)` function explicitly handles `latest`, `now`, `..`, and `../..`, and the `start/end` split. So the spec requirement is acknowledged in the codebase — it's just not on the path that runs for incoming HTTP query parameters.

## Mapping spec → defect

| Spec requirement                                         | cs-go observation endpoint behaviour |
|----------------------------------------------------------|--------------------------------------|
| Interval `start/end` filters server-side                 | Not applied — single `[]string` element with `/` is fed straight into `time.Parse(RFC3339, ...)` and silently fails |
| Open-start `../end` and open-end `start/..`              | Not applied — same parse failure as above |
| Keyword `latest` returns only most recent observation    | Not applied — keyword path only exists in `ToTimeRange(string)`, never reached from the handler |
| Invalid value (e.g. `frobnicate`) → HTTP 400              | Returns HTTP 200 with full unfiltered page |

All four bullet rows match the live-test results, which match the static-analysis prediction.
