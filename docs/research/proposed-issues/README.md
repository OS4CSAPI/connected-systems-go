# Proposed issues — drafts pending filing on the GitHub tracker

This directory holds **issue bodies prepared for filing** against
`OS4CSAPI/connected-systems-go`. Each `*.md` here is a fully-formed issue body
ready to copy into the GitHub "New issue" form (or to file via the REST API).

## Why drafts live in the repo first

Findings surfaced during the issue-evaluation work (see
[`docs/research/issue-evaluations/`](../issue-evaluations/)) sometimes turn up
new defects that are out of scope for the issue currently being evaluated. To
avoid losing them, we draft the new issue body in this folder, commit it, and
then file the GitHub issue from this body. The committed draft becomes the
audit trail.

## Template

These drafts adapt the best practices from the OS4CSAPI ogc-client-CSAPI_2
[Phase 8 issue-creation template](https://github.com/OS4CSAPI/ogc-client-CSAPI_2/blob/phase-8/docs/governance/issue-creation-prompt-template-phase-8.md)
to this repo's discovery-mode context (no locked-decision trio, no Phase 8 task
IDs). Specifically we keep:

- Strong header (severity, category, source, ownership)
- **Automated acceptance gates only** — no manual-review checkboxes
- Three-step **Definition of Done / closing workflow**
- Mandatory **Dependencies** / related-issues block
- **Scope — what NOT to touch** fences
- **References** table with line-precise code links and spec citations
- **Filing checklist** as the last section

We drop:

- Phase 8 task ID / "Locked Decision" sections (we are in discovery mode)
- The general-purpose "Proposed Solutions Option A/B" block in favour of a
  short **Suggested fix** subsection inside Problem Statement; the maintainer
  retains decision authority.

## Index

| File | Issue | Title (short) | Severity | Discovered during |
|---|---|---|---|---|
| [`stray-uid-silently-dropped.md`](./stray-uid-silently-dropped.md) | [#13](https://github.com/OS4CSAPI/connected-systems-go/issues/13) | `POST /systems/{id}/datastreams` silently drops stray top-level `uid` field | P2-Important | issue #1 evaluation §2.8 T6 |
| [`api-endpoint-is-stub.md`](./api-endpoint-is-stub.md) | [#14](https://github.com/OS4CSAPI/connected-systems-go/issues/14) | `GET /api` returns 86-byte stub with no paths or schemas | P2-Important | issue #1 evaluation §2.8 T2 |
| [`systems-field-leaks-into-json.md`](./systems-field-leaks-into-json.md) | [#15](https://github.com/OS4CSAPI/connected-systems-go/issues/15) | Capital-`S` `Systems` GORM relationship field leaks into every datastream JSON response | P3-Minor | issue #1 evaluation §2.8 T7 |

## Filing workflow

1. Open the draft `.md` here.
2. Copy the body **starting from the first line of the front-matter table**
   (severity, category, source) into the GitHub "New issue" form.
3. Apply the labels listed in the draft's `Labels:` field.
4. Once filed, edit this draft to add the resulting issue number/URL at the top
   and commit the update so the audit trail is closed.
