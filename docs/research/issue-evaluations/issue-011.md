# Issue #11 — Evaluation

**Issue:** [#11](https://github.com/OS4CSAPI/connected-systems-go/issues/11) — Temporal query parameters (resultTime, phenomenonTime, datetime) silently ignored — all values discarded, including `latest`
**Reporter framing:** P2-Important, bug + code-review, escalated from P3 after empirical re-test.
**Repo @:** `175f263` (`origin/main`).
**Endpoint:** `https://129-80-248-53.sslip.io/csapi-go-head` (Date header `Thu, 30 Apr 2026 22:47:41 GMT`).

## Verdict

**VALIDATED** — real defect. P2-Important is the right severity; a P1 framing is defensible because temporal filtering is foundational CSAPI Part 2 capability and the failure is silent (HTTP 200 with full data instead of either a filtered page or an HTTP 400).

## Static analysis (summary)

Full notes: [evidence/issue-011/static-analysis-2026-04-30.md](../evidence/issue-011/static-analysis-2026-04-30.md)

The handler does parse `resultTime` and `phenomenonTime` and the repository does have a SQL `WHERE` builder for them (`observation_repository.go::applyFilters` lines 109–127). So the issue's "the entire parser is a no-op" framing is too strong — the wiring exists end-to-end. The defect is isolated to **one dispatch line** in [internal/model/common_shared/time_range.go](../../../internal/model/common_shared/time_range.go) `ParseTimeRange`:

- Handler invokes `ParseTimeRange(vals)` where `vals` is `[]string` (from `r.URL.Query()[name]`).
- `ParseTimeRange` switches on type and routes `[]string` → `ToTimeRangeFromSlice`.
- `ToTimeRangeFromSlice` parses each element as RFC 3339 directly. It does **not** split on `/` and does **not** handle the keywords `latest`, `now`, `..`, `../..`.
- The richer `ToTimeRange(string)` function — which does both splitting and keyword recognition — is never reached for HTTP query parameters.
- `time.Parse` errors are silently dropped (`if err == nil { ... }`); on any failure both `Start` and `End` stay `nil`.
- `applyFilters` then adds no `WHERE` clause when both bounds are `nil` and reports no error.

Net behaviour: every value of `resultTime` / `phenomenonTime` produces an empty `TimeRange{}`, the SQL is unfiltered, and the response is HTTP 200 with the default page.

## Live verification (summary)

Full notes: [evidence/issue-011/live-test-2026-04-30.md](../evidence/issue-011/live-test-2026-04-30.md)

Three observations seeded with `result_time` in 2024, 2025, 2026. Nine probes covering: `latest`, `latest&limit=1`, future-window 2099 (zero matches expected), garbage `frobnicate`, narrow valid window for one obs, `phenomenonTime=now`, narrow valid `phenomenonTime` window, open-start interval, open-end interval. All nine return HTTP 200 with all three observations. Only `latest&limit=1` happens to return one item — by accident of `LIMIT 1` applied to an unfiltered `result_time DESC`-sorted page. After testing, the seeded observations were deleted (HTTP 204 each).

## Spec authority (summary)

Full notes: [evidence/issue-011/spec-authority-2026-04-30.md](../evidence/issue-011/spec-authority-2026-04-30.md)

CSAPI Part 2 §13.3 defines `resultTime` and `phenomenonTime` with OGC API – Features Part 1 §7.15.4 interval grammar. CSAPI Part 2 §13.3.2 statement D normatively defines the `latest` keyword for `resultTime` on observation collections. Features Part 1 §7.15.4 also requires HTTP 400 for invalid values.

## Refinement to issue framing

The issue claims "the entire temporal parameter parser is a no-op". A precise statement is: **the parse-side dispatch in `ParseTimeRange` routes HTTP query parameters to a slice helper that does not handle interval syntax, open-bound `..`, or any of the spec keywords. The downstream handler / repository / SQL plumbing is correct.** This is a one-function fix. The empirical user-visible behaviour the reporter described (every value silently discarded, garbage produces 200) is exactly what the code does, so the practical impact described in the issue holds.

## Suggested fix (sketch — not implemented in this evaluation)

In `observation_query_params.go::BuildFromRequest`, instead of:

```go
tr := common_shared.ParseTimeRange(vals)
```

call the string-aware path on the joined value:

```go
tr := common_shared.ToTimeRange(strings.Join(vals, "/"))
```

…or change `ToTimeRangeFromSlice` to detect a single-element slice that contains `/` and/or a keyword, and delegate to `ToTimeRange(string)`. Either way: `time.Parse` errors should bubble up to the handler so the API can return HTTP 400 on garbage input rather than silently degrading to "no filter".

A complete fix should also extend coverage to `datetime` on systems / deployments / sampling-features collections per OGC 23-001r0 §8.7 — that scope is mentioned in the issue but was not empirically probed in this evaluation.

## Severity

P2-Important is firm. P1 is defensible: a temporal-filter failure that returns HTTP 200 with a full page silently mis-represents server state to clients. Down-stream agents that, e.g., poll `?resultTime=latest` every minute will receive `limit`-default rows of stale data and have no signal that the filter never executed.

## Recommendations to issue thread

- Acknowledge the empirical claim is reproduced.
- Narrow the framing to: *parse-side dispatch in `ParseTimeRange`*, not *the entire parser*.
- Note the silent-failure aspect (HTTP 200 instead of 400 on garbage) as an additional spec-conformance defect distinct from the missing-filter defect — both are caused by the same parse path swallowing errors.
- Suggest the fix is small and localised to `time_range.go` + a one-line change in each affected `*_query_params.go::BuildFromRequest`.
