---
name: api-contract-writer
description: >
  Use this skill when the user wants to derive API contract specifications from reviewed use cases and technical design. Triggers include 'write the API contract', 'define the endpoints', 'derive the REST API', 'what endpoints do we need', 'produce the API spec', 'map use cases to HTTP'. Also use in the agentic pipeline during Phase 3B. Requires READY use cases and technical design output (interactors, typed errors, domain events). Do NOT use before technical design exists.
---

# API Contract Writer

Derives HTTP API contract specifications from reviewed use cases and their technical design output.

## Overview

Interactors return typed results and typed errors. HTTP endpoints must translate those into status codes, response bodies, and error envelopes. Without an explicit contract, developers invent ad-hoc mappings and frontends guess at error shapes. The contract specifies every response and traces it to the use case element it implements.

## Dependencies

1. **Reviewed use cases** — READY status from use-case-reviewer
2. **Technical design** — from use-case-to-technical-design: interactors, typed errors, domain events
3. **QA scenarios** (if any) — rate limiting, auth requirements, SLAs

## Core Mapping: Use Case Element to HTTP Construct

| Use case element | HTTP construct | Example |
|---|---|---|
| Goal (verb + noun) | HTTP method + resource URL | "Submit form" -> POST /forms/{formId}/submissions |
| Preconditions (state checks) | 4xx error responses | FormNotPublished -> 404 |
| MSS postconditions | Success response body + status | Record created -> 201 { recordId, status } |
| Extensions — validation | 422 with error envelope | RequiredFieldMissing -> 422 { errors } |
| Extensions — invariant conflict | 409 Conflict | DuplicateSubmission -> 409 |
| Extensions — alternate success | 2xx with different status | Approval mode -> 202 { status: "pending" } |
| QA scenarios | Auth/rate-limit headers | QA-03 -> Rate-Limit headers, 429 response |
| Domain events | Webhook payloads | RecordCreated -> webhook POST |

## API Contract Template

One contract block per endpoint:

```
Endpoint: [METHOD] /v1/[resource-path]
Implements: [UC-ID] -- [Use case title]
Actor: [Primary actor] ([public | authenticated | role])
Auth: [None | Bearer token | API key] (ref QA scenario)

Request:
  Path params: [name (type, required)]
  Query params: [name (type, optional, default)]
  Body: { [field]: [type], ... }

Response -- success:
  [status] [reason] -> { [field]: [type], ... }
  Derived from: [UC-ID] postconditions

Response -- errors:
  [status] [reason] -> { [error envelope] }
  Derived from: [UC-ID] ext [N][letter] ([ErrorType])

Pagination: [strategy or N/A]
Rate limiting: [constraint from QA scenario or "none specified"]
```

## Guidance

### HTTP method selection

| Goal verb | Method |
|---|---|
| Create, submit, send | POST |
| Read, view, list, get | GET |
| Update, edit, rename | PUT (full) or PATCH (partial) |
| Delete, remove, archive | DELETE |
| Toggle, activate | PATCH on sub-resource |

### URL structure

- Plural nouns: `/forms`, `/records`, `/workspaces`
- One nesting level max: `/forms/{formId}/submissions`
- Non-CRUD actions use verb sub-resource: `POST /forms/{formId}/publish`
- IDs in path params, filters in query params

### Error envelope format

```json
{
  "error": "machine_readable_code",
  "message": "Human-readable description",
  "errors": [
    { "field": "fieldId", "code": "required", "message": "This field is required" }
  ]
}
```

Top-level `error` for single errors (404, 409, 403). `errors` array for validation (422) with per-field detail. Codes are snake_case of the typed error class.

### Pagination

List endpoints: cursor-based `?cursor=X&limit=N`, response `{ data: [...], nextCursor: string | null }`, default 25, max 100.

### Versioning

URL prefix `/v1/`. New version only on breaking changes.

## How QA Scenarios Feed In

QA scenarios constrain the contract without changing domain logic:

| QA scenario type | Contract impact |
|---|---|
| Rate limiting (QA-03) | `X-RateLimit-*` headers; 429 response |
| Authentication | `Auth:` field; 401/403 error responses |
| Response time SLA | Note for infrastructure sizing |
| Data residency | Geographic routing note |

Each constraint references its scenario ID in the contract.

## Output Format

For each use case with an HTTP-facing adapter, produce:

1. The contract block (template above)
2. Response type definition (TypeScript or Go, matching technical design)
3. Error-mapping table: DomainError -> HTTP status + error code

### Verification Checklist

- [ ] Every interactor result field appears in the success response
- [ ] Every typed domain error maps to one HTTP status + error code
- [ ] Every use case extension has a corresponding error response
- [ ] Precondition failures are 4xx (not 5xx)
- [ ] Auth and rate limiting reference QA scenario IDs
- [ ] URLs use plural nouns, one nesting level max
- [ ] Error envelope format is consistent across endpoints
- [ ] Pagination specified for every list endpoint
- [ ] Each response includes a "Derived from" trace

## Common Mistakes

- **Inventing responses not in the use case.** Every response traces to a postcondition, extension, or QA scenario. No source, no response.
- **Generic error codes.** `"bad_request"` is useless. Use the typed error name: `"required_field_missing"`.
- **Mapping all errors to 400.** Validation is 422, not-found is 404, conflicts 409, permissions 403.
- **Forgetting alternate success responses.** Approval mode returning 202 instead of 201 needs its own block.
- **Omitting QA-driven responses.** 429 and 401/403 are part of the contract.
- **Deep URL nesting.** One level max. Flatten with top-level resources.
