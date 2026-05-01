# Issue #14 — Spec-authority evidence

**Date:** 2026-04-30
**Question:** Which normative requirements (if any) does the 88-byte
`GET /api` stub violate, given (a) the OAS 3.0 media type advertised on
the response, and (b) the conformance classes cs-go declares at
`/conformance`?

---

## 1. OpenAPI Specification 3.0.3 — required root field `paths`

Source: <https://spec.openapis.org/oas/v3.0.3> §4.7.1.1 *Fixed Fields* of
the OpenAPI Object.

> | Field name | Type | Description |
> |---|---|---|
> | `openapi` | string | **REQUIRED.** … |
> | `info` | Info Object | **REQUIRED.** … |
> | `servers` | [Server Object] | … |
> | `paths` | Paths Object | **REQUIRED.** The available paths and operations for the API. |

The specification marks `paths` as **REQUIRED** for the root OpenAPI
Object in OAS 3.0.x. (In OAS 3.1, `paths` was relaxed to
optional-when-`webhooks`-or-`components`-is-present; that relaxation
does not apply here because cs-go's response declares
`"openapi": "3.0.0"` and `Content-Type:
application/vnd.oai.openapi+json;version=3.0`.)

The cs-go body is:

```json
{"openapi": "3.0.0", "info": {"title": "OGC Connected Systems API", "version": "1.0.0"}}
```

This document **omits the required `paths` field**. It is therefore
**not a valid OpenAPI 3.0 document** by the OAS spec's own root-object
schema. A client that validates the response against the OAS 3.0 schema
will reject it.

## 2. OGC API – Common Part 1 (OGC 19-072) — `landing-page` requirements

Source: <https://docs.ogc.org/is/19-072/19-072.html>

cs-go declares conformance to the `landing-page` requirements class via
its `/conformance` document
(`http://www.opengis.net/spec/ogcapi-common-1/1.0/conf/landing-page`).
This makes the following requirements binding on cs-go:

### 2.1 `/req/landing-page/api-definition-op` (§9.2.1)

> A. The server SHALL support the HTTP GET operation on all links from
> the landing page that have the relation type `service-desc`.
> …
> C. The responses to all HTTP GET requests issued in A and B SHALL
> satisfy requirement `/req/landing-page/api-definition-success`.

cs-go's landing page advertises `/api` as `service-desc`; the server
returns HTTP 200 for `GET /api`, so part A passes operationally.
Part C delegates correctness to the next requirement.

### 2.2 `/req/landing-page/api-definition-success` (§9.2.2) — VIOLATED

> A. A successful execution of the operation SHALL be reported as a
> response with a HTTP status code 200.
> **B. The content of that response SHALL be an API Definition document.**
> C. The API Definition document SHALL be consistent with the media type
> identified through HTTP content negotiation.

Statement B is a strict `SHALL`. The 88-byte stub:

- claims to be an OpenAPI 3.0 document via its `Content-Type` header;
- but is not a valid OpenAPI 3.0 document (missing required `paths`,
  see §1 above);
- describes none of the API's actual operations (Features, Datastreams,
  Observations, Systems, etc. — all routes the server exposes);
- therefore cannot serve the function of an "API Definition document"
  in the sense of OGC API – Common Part 1 §9.2 (whose purpose, per the
  surrounding text, is to "be used by client developers to understand
  the supported services, by software clients to connect to the server,
  and by development tools to support the implementation of servers and
  clients").

This is a **direct, binding `SHALL` violation** caused by cs-go's own
declaration of `conf/landing-page`.

Statement C is also violated in spirit: the `Content-Type` declares
`application/vnd.oai.openapi+json;version=3.0` but the body is not a
valid instance of that media type (per §1).

### 2.3 `/rec/landing-page/api-definition-oas` (§9.2.2) — VIOLATED (SHOULD)

> A. If the API definition document uses the OpenAPI Specification 3.0,
> THEN The document SHOULD conform to the OpenAPI Specification 3.0
> requirements class.

cs-go's document declares `"openapi": "3.0.0"` and serves it under the
OAS 3.0 media type, so the antecedent applies. The document does not
conform to the OAS 3.0 requirements class (see §3 below). This is a
`SHOULD` violation.

## 3. OGC API – Common Part 1 §11 — OpenAPI 3.0 requirements (informational)

cs-go does **not** declare `/conf/oas30`, so the requirements in §11 are
**not strictly binding** on cs-go. They are documented here because:

- the recommendation `/rec/landing-page/api-definition-oas` (§2.3 above)
  invites these as the conformance target the document "should" meet;
- the issue body quotes them; we record their text accurately for the
  evaluation record.

### `/req/oas30/oas-definition-2` (§11.1)

> A. The JSON representation SHALL conform to the OpenAPI Specification,
> version 3.0.

### `/req/oas30/completeness` (§11.2)

> A. The OpenAPI definition SHALL specify for each operation all HTTP
> Status Codes and Response Objects that the API uses in responses.
> B. This includes the successful execution of an operation as well as
> all error situations that originate from the server.

### `/req/oas30/oas-impl` (§11.1)

> A. The API SHALL implement all capabilities specified in the OpenAPI
> definition.

A stub document with no `paths` cannot satisfy `oas-definition-2`
(invalid OAS 3.0) or `completeness` (no operations described).
`oas-impl` is trivially "satisfied" only because the document describes
nothing.

If cs-go *were* to add `/conf/oas30` to its conformance declaration
(or if a future Connected Systems profile mandates it transitively),
these become hard `SHALL` violations.

## 4. Connected Systems Part 1 / Part 2 — `api-common`

cs-go's `/conformance` declares both
`ogcapi-connectedsystems-1/1.0/conf/api-common` and
`ogcapi-connectedsystems-2/1.0/conf/api-common`. The Connected Systems
`api-common` conformance class transitively depends on OGC API – Common
Part 1 (it is the foundation referenced by the CSAPI specifications).
Examination of the CSAPI specs indicates the `api-common` class
incorporates the landing-page requirements class by reference; this
strengthens the binding state of `/req/landing-page/api-definition-success`
(it is invoked twice: once directly via the OGC API – Common
declaration, and once transitively via the CSAPI declarations).

## Summary of binding violations

| Requirement | Status | Strength | Source of binding |
|---|---|---|---|
| OAS 3.0.3 §4.7.1.1 (paths REQUIRED) | Violated | Hard schema rule | Self-asserted via `Content-Type` header and `"openapi":"3.0.0"` |
| `/req/landing-page/api-definition-success` B | Violated | `SHALL` | cs-go declares `conf/landing-page` |
| `/req/landing-page/api-definition-success` C | Violated | `SHALL` | same |
| `/rec/landing-page/api-definition-oas` | Violated | `SHOULD` | same |
| `/req/oas30/oas-definition-2` | Would be violated | `SHALL` if asserted | cs-go does **not** declare `conf/oas30` (informational) |

The defect is real, has spec-citable severity, and rests on at least one
strict `SHALL` requirement that cs-go has self-bound to via its own
conformance declaration.
