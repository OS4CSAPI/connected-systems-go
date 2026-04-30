# Issue #11 — Static analysis (2026-04-30)

Repo @ HEAD = `175f263` (origin/main).

## Files reviewed

- [internal/model/query_params/query_params.go](../../../../internal/model/query_params/query_params.go) (base `QueryParams`)
- [internal/model/query_params/observation_query_params.go](../../../../internal/model/query_params/observation_query_params.go)
- [internal/model/common_shared/time_range.go](../../../../internal/model/common_shared/time_range.go)
- [internal/api/observation_handler.go](../../../../internal/api/observation_handler.go)
- [internal/repository/observation_repository.go](../../../../internal/repository/observation_repository.go)

## Code path

1. **Handler** `observation_handler.go` lines 42–45 (`ListObservations`) and 65–75 (`ListDatastreamObservations`):

   ```go
   params := queryparams.ObservationsQueryParams{}.BuildFromRequest(r)
   observations, total, err := h.repo.List(params, nil)        // or ListByDatastream(...)
   ```

2. **Parser** `observation_query_params.go` lines 46–55:

   ```go
   if vals := r.URL.Query()["phenomenonTime"]; len(vals) > 0 {
       tr := common_shared.ParseTimeRange(vals)                 // vals is []string
       params.PhenomenonTime = &tr
   }
   if vals := r.URL.Query()["resultTime"]; len(vals) > 0 {
       tr := common_shared.ParseTimeRange(vals)                 // vals is []string
       params.ResultTime = &tr
   }
   ```

3. **Dispatch** `time_range.go` `ParseTimeRange(value interface{})`:

   ```go
   case []string:
       return ToTimeRangeFromSlice(v)                            // <-- called for r.URL.Query()[...]
   case string:
       return ToTimeRange(v)                                     // <-- has keyword + interval handling, NEVER reached from handler
   ```

4. **`ToTimeRangeFromSlice([]string)`** lines 138–163:

   ```go
   if parts[0] != "" && parts[0] != ".." {
       if t, err := time.Parse(time.RFC3339, parts[0]); err == nil {  // err silently ignored
           startTime = &t
       }
   }
   if len(parts) > 1 {
       if parts[1] != "" && parts[1] != ".." {
           if t, err := time.Parse(time.RFC3339, parts[1]); err == nil {
               endTime = &t
           }
       }
   }
   ```

   This function:
   - Does **not** split a single `"start/end"` string on `/`. The OGC interval form `?resultTime=2024-01-01T00:00:00Z/2024-01-31T23:59:59Z` arrives as `vals = []string{"2024-01-01T00:00:00Z/2024-01-31T23:59:59Z"}`. `time.Parse(RFC3339, ...)` fails on the slash → `startTime` stays `nil`.
   - Does **not** recognise the spec-defined keywords `latest`, `now`, `..`, `../..` — those are only matched in `ToTimeRange(string)`, which is not reached from this code path. `time.Parse(RFC3339, "latest")` fails silently → `nil`.
   - Silently swallows `time.Parse` errors (`if err == nil` only sets the value, never reports), so any unparseable input — including garbage like `frobnicate` — produces `TimeRange{Start: nil, End: nil}` rather than an HTTP 400.

5. **Filter application** `observation_repository.go` lines 86–127 (`applyFilters`) is correctly wired and is reached from both `List` and `ListByDatastream`. But the only conditions are:

   ```go
   if params.ResultTime != nil {
       if params.ResultTime.Start != nil && params.ResultTime.End != nil { ... query.Where(...) }
       else if params.ResultTime.Start != nil                              { ... query.Where(...) }
       else if params.ResultTime.End != nil                                { ... query.Where(...) }
   }
   ```

   When both `Start` and `End` are `nil`, **no `WHERE` clause is added** and no error is raised. Same for `params.PhenomenonTime`.

## Defect

`ParseTimeRange` is dispatched on `[]string` (from `r.URL.Query()[name]`) and routed to `ToTimeRangeFromSlice`, which does neither interval-string splitting nor keyword recognition. The richer `ToTimeRange(string)` path — which does both — is never reached for HTTP query parameters. The handler/repository plumbing downstream is correct; the bug is isolated to the parse dispatch.

Net effect: every value of `resultTime` and `phenomenonTime` on `/observations` and `/datastreams/{id}/observations` produces an empty range, no SQL `WHERE` clause is added, and the unfiltered default page is returned with HTTP 200. This matches the empirical behaviour reported in the issue.

## Notes

- Other resources have analogous code paths but are not in scope for this issue's empirical claims; they would need separate verification (the issue explicitly notes systems / deployments / sampling features per OGC 23-001r0 §8.7).
- `internal/repository/observation_repository.go` has not changed since its first commit `f2cf1c3` (`git log` confirms), so the deployed `csapi-go-head` binary contains exactly this code.
