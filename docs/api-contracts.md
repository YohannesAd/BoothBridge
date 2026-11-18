# Initial REST API contracts

Status: proposed contract; no endpoints or OpenAPI artifact exist yet. Base path: `/api/v1`. The examples are synthetic. Spring Boot owns this API; Next.js consumes it on the same origin.

## Common rules

- JSON field names use camelCase; identifiers are UUID strings. UTC instants use RFC 3339 strings ending in `Z`, dates use `YYYY-MM-DD`, and event zones use IANA identifiers such as `America/Chicago`.
- All business endpoints require a session. Resolve the owner from the principal, never a client `ownerId`. Verify ownership on every event, resource, filter, and referenced ID. An absent record, a record owned by another user, or a mismatched scoped reference returns `404` without confirming its existence.
- `GET` is read-only. Creation returns `201`, a `Location` header, and the representation. Reads/updates return `200`; successful deletion and logout return `204` with no body. Authenticated and CSRF responses use `Cache-Control: no-store`.
- Single-resource create/read/update responses include a numeric `version` and `ETag: "<version>"`. `PATCH` and resource `DELETE` require that strong ETag in `If-Match`; missing is `428`, malformed is `400`, stale is `412`. Compare atomically with JPA optimistic locking, including deletion. Successful changes advance the version. Logout is exempt.
- Collections use `page` (zero-based, default 0) and `size` (default 20, maximum 100). Reject invalid ranges. Return `{ "items": [], "page": 0, "size": 20, "totalElements": 0 }`. Default order is `createdAt DESC, id DESC`; task lists also use this order in v1. Concurrent inserts may shift offset pages.
- Send `Content-Type: application/json` for bodies. `PATCH` uses partial DTO updates: omitted fields stay unchanged, null clears only optional fields, and required fields cannot be null. Reject empty patches, unknown fields, server-managed fields, and invalid enum values. Request bodies are limited to 64 KiB.
- Trim surrounding whitespace on short text and URLs, then validate; retain note formatting. See the [URL policy](security.md#qr-and-saved-url-policy) for parsing and canonicalization. Errors must not echo contact data or full URLs.
- Unsafe methods, including logout, require the current CSRF token header. See [security.md](security.md) for the session and sign-in flow.

## Identity and session

| Method/path | Behavior |
| --- | --- |
| `GET /auth/csrf` | Available before sign-in; returns `headerName` and `token` for the current browser context. No caching. Refresh after sign-in, session expiry, or logout. |
| `GET /me` | Returns the signed-in account's `id` and nullable `displayName`; `401` if signed out. Never expose provider tokens or subject identifiers. |
| `POST /auth/logout` | Requires session and CSRF token; invalidates the session and expires its cookie. Returns `204`. |

Provider-managed OIDC sign-in uses Spring Security's browser routes outside `/api/v1`: `/oauth2/authorization/{registrationId}` and `/login/oauth2/code/{registrationId}`. Only configured registrations are valid. Their redirects are part of sign-in, not business API responses. There is no local password, registration, JWT issuance, or QR redirect endpoint. Provider selection is pending.

## Resource endpoints

For each collection in the table, support `GET` (list) and `POST` (create). For each item, support `GET`, `PATCH`, and `DELETE`. All paths below are relative to `/api/v1`.

| Resource | Collection path | Item suffix | Additional list filters |
| --- | --- | --- | --- |
| Event | `/events` | `/{eventId}` | None |
| Company | `/events/{eventId}/companies` | `/{companyId}` | None |
| Saved link | `/events/{eventId}/links` | `/{linkId}` | `companyId`, `opportunityId`, `category` |
| Contact | `/events/{eventId}/contacts` | `/{contactId}` | `companyId`, `kind` |
| Opportunity | `/events/{eventId}/opportunities` | `/{opportunityId}` | `companyId`, `applicationStatus` |
| Note | `/events/{eventId}/notes` | `/{noteId}` | `companyId` |
| Follow-up task | `/events/{eventId}/tasks` | `/{taskId}` | `companyId`, `status`, `dueBefore` (inclusive UTC instant) |

Filters combine with AND. Missing `companyId` includes company-associated and event-only records where supported. `dueBefore` excludes tasks with no due time. Referenced IDs in filters must be owned and in scope; malformed filters return `400`, mismatched references return `404`. Unknown query parameters are rejected.

Two dedicated operations keep workflow states explicit. Their state is included in the parent resource response; these are not separate stored resources.

| Method/path | Request fields | Behavior |
| --- | --- | --- |
| `PATCH /events/{eventId}/companies/{companyId}/check-in` | Required `status` | Set `NOT_CHECKED_IN` or `CHECKED_IN`; return the updated company. Timestamp rules are in [database design](database-design.md#controlled-values-and-invariants). |
| `PATCH /events/{eventId}/opportunities/{opportunityId}/application` | Required `status`; optional nullable `appliedOn` | Set one of the application states; return the updated opportunity. Omitting `appliedOn` preserves it except `NOT_APPLIED`, which clears it. A nonnull date with `NOT_APPLIED` is rejected. |

Both operations require `If-Match` and CSRF. Opening, saving, or categorizing a link calls neither operation. No endpoint checks whether a third-party application or check-in actually completed.

## Payloads and validation

All resource responses include `id`, `createdAt`, `updatedAt`, and `version`; event children also include `eventId`. Responses include the fields below, using null for absent optional values. Never accept IDs, audit fields, version, or ownership as writable body properties.

| Resource | Create payload | Editable through item PATCH | Additional response fields |
| --- | --- | --- | --- |
| Event | Required `name`, `startDate`, `endDate`, `timeZone`; optional `location` | Same fields | None |
| Company | Required `name`; optional `boothLocation` | Same fields | `checkIn: {status, recordedAt}`; initially `NOT_CHECKED_IN`, null |
| Saved link | Required `companyId`, `label`, `url`, `category`, `source`; optional `opportunityId` | `label`, `url`, `category`, `opportunityId` | None; `source` and `companyId` remain fixed |
| Contact | Required `name`, `kind`; optional `companyId`, `role`, `email`, `phone` | Same fields | None |
| Opportunity | Required `companyId`, `title`; optional `location` | `title`, `location` | `application: {status, appliedOn}`; initially `NOT_APPLIED`, null |
| Note | Required `body`; optional `companyId` | Same fields | None |
| Task | Required `title`; optional `companyId`, `dueAt` | Same fields plus `status` | `status` initially `OPEN`; nullable `completedAt` set/cleared by the server |

Parent event IDs are immutable. Required company IDs on links/opportunities are immutable. Optional associations can change only within the same owned event; an opportunity reference must match a link's fixed company.

- Required short text must be nonblank. Limits: names/labels/roles 120; company booth location 80; event/opportunity location and opportunity/task title 160; email 254; phone 40; time zone 64; note body 10,000; URL 2,048 characters. A note must contain non-whitespace text; its line breaks are preserved.
- Event date ranges allow past events but require `endDate >= startDate`. Time zones must be recognized IANA zones. Application dates cannot be in the future relative to the event's time zone. Future and past task due times are both valid.
- Contact email is optional; a provided value must pass syntactic email validation. Do not verify ownership or require uniqueness. Phone is optional display text, not a number or a claim of a verified phone.
- All statuses and link/contact enums are defined in [database design](database-design.md#controlled-values-and-invariants). Do not accept arbitrary strings or numeric ordinals.
- Company check-in and application states can be corrected directly. Generic company/opportunity PATCH cannot write their workflow states. Generic task PATCH may complete or reopen a task using the documented timestamp rules.
- URL validation is authoritative on the server, runs again on edit, and never fetches the destination. Duplicate URLs are permitted. Pause scanner results and disable pending form submission in the UI; POST retries are not idempotent in v1. After an uncertain network result, refresh the list before retrying to avoid accidental duplicates.
- Event/company deletion fails with `409 DEPENDENT_RESOURCES` while dependent records exist. Opportunity deletion fails while a saved link references it. Optional references can be cleared first; there is no silent cascade.

## Save-link example

Illustrative `POST /api/v1/events/{eventId}/links` body, with session and CSRF headers provided at runtime:

```json
{
  "companyId": "aaaaaaaa-aaaa-4aaa-8aaa-aaaaaaaaaaaa",
  "label": "Engineering roles",
  "url": "https://careers.example.com/jobs?source=conference",
  "category": "CAREERS",
  "source": "QR",
  "opportunityId": null
}
```

The `201` response includes the saved fields, event ID, new resource ID, audit fields, `version: 0`, and `ETag: "0"`. `Location` points to its item endpoint. The frontend shows success and an explicit Open destination action; the response never redirects to that URL.

## Error handling

Use `application/problem+json` following [RFC 9457](https://www.rfc-editor.org/info/rfc9457/). Use `type: "about:blank"`, the HTTP status phrase as `title`, and extensions `code`, `traceId`, and optional field `errors`. Clients branch on HTTP status and `code`, not human-readable prose. `instance` is the request path without its query string. Security filters must use the same envelope for API failures.

```json
{
  "type": "about:blank",
  "title": "Bad Request",
  "status": 400,
  "detail": "One or more fields are invalid.",
  "instance": "/api/v1/events/bbbbbbbb-bbbb-4bbb-8bbb-bbbbbbbbbbbb/links",
  "code": "VALIDATION_FAILED",
  "traceId": "example-trace-id",
  "errors": [
    {"field": "url", "code": "HTTPS_REQUIRED", "message": "Use an absolute HTTPS URL."}
  ]
}
```

| HTTP status | Stable codes / meaning |
| --- | --- |
| `400` | `MALFORMED_REQUEST`, `VALIDATION_FAILED`, `INVALID_QUERY`: syntax, fields, body rules, or query parameters |
| `401` | `AUTHENTICATION_REQUIRED`: session absent/expired; API does not redirect to an HTML login page |
| `403` | `CSRF_INVALID` or `ACCESS_DENIED`: invalid/missing CSRF for a valid session, or an operation-level denial |
| `404` | `NOT_FOUND`: missing or inaccessible scoped resource/reference |
| `409` | `DEPENDENT_RESOURCES`: deletion blocked by references |
| `412` | `STALE_VERSION`: conditional write lost a race; reload before reapplying edits |
| `413` | `PAYLOAD_TOO_LARGE`: request limit exceeded |
| `415` | `UNSUPPORTED_MEDIA_TYPE`: body format unsupported |
| `428` | `PRECONDITION_REQUIRED`: resource update/delete lacks `If-Match` |
| `429` | `RATE_LIMITED`: throttled request; include `Retry-After` |
| `500` | `INTERNAL_ERROR`: generic failure with trace ID, no exception internals |

For protected business paths, absent authentication takes precedence over write precondition and payload validation errors. Authentication/authorization must precede disclosure of resource-specific details. Expected field failures can explain the rule but never expose SQL, stack traces, rejected private values, or another user's metadata. Rate limiting and these handlers must be implemented and tested; they do not exist yet.
