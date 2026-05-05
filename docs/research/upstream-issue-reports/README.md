# Upstream Issue Reports

Per-issue **synthesis reports** produced under Phase 4 Step 4.2 of
[`../upstream-followup-plan.md`](../upstream-followup-plan.md).

Each report is the durable internal record of an upstream filing:
findings, alternatives considered, fix recommendation, re-verification
log, and a public-facing extract that becomes the upstream issue body.

## Layout

```
docs/research/
  upstream-issues/
    plan-NN-<slug>.md        <- research plan (sources to review)
  upstream-issue-reports/
    report-NN-<slug>.md      <- synthesis report (THIS folder)
```

`NN` and `<slug>` match across the plan and report for the same item.

## Mandatory pre-work

Before authoring or working from any report, re-read the curated
authoritative-references list at:

<https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/research/references.md>

Self-sourcing spec/standard references is forbidden. Gaps must be
surfaced to the user before drafting.

## Required report structure

See Step 4.2 of `../upstream-followup-plan.md` for the full required
contents. Summary:

1. Header block (backlog item, source fork issues, plan path, date)
2. Re-verification record (commands + output + `upstream/main` SHA)
3. Static evidence
4. Live evidence (if applicable)
5. Spec authority (citations traceable to the references list)
6. Alternatives considered (internal-only)
7. Recommended fix
8. Scope guard
9. Fork-side context (internal-only)
10. Open questions resolved
11. **Public-facing extract** — verbatim source for the upstream issue body
</content>
</invoke>