# API Design Standards

These conventions apply to every HTTP API in the Whirlwind platform. Consistency here is what makes services feel like one platform rather than N separate projects.

---

## URL structure

```
/{version}/{resource}/{id}/{sub-resource}
```

- Lowercase, hyphen-separated: `/sms-messages` not `/smsMessages`
- Plural nouns for collections: `/messages` not `/message`
- Version prefix on all routes: `/v1/messages`
- IDs in the path: `/v1/messages/{id}`

```
GET    /v1/messages           # list
POST   /v1/messages           # create
GET    /v1/messages/:id       # get one
PATCH  /v1/messages/:id       # partial update
DELETE /v1/messages/:id       # delete

POST   /v1/messages/:id/hide  # actions that aren't CRUD
```

---

## Request format

- `Content-Type: application/json` on all requests with a body
- Request bodies are JSON objects — never arrays at the root level
- All timestamps in ISO 8601 UTC: `2026-04-25T01:00:00Z`
- Phone numbers in E.164 format: `+12345678901`
- IDs are UUIDs v4

---

## Response format

All responses use a consistent envelope:

**Success (single resource):**
```json
{
  "data": { ... }
}
```

**Success (collection):**
```json
{
  "data": [ ... ],
  "meta": {
    "total": 142,
    "page": 1,
    "per_page": 25
  }
}
```

**Error:**
```json
{
  "error": {
    "code": "validation_failed",
    "message": "Human-readable description",
    "details": [
      { "field": "phone_number", "message": "must be in E.164 format" }
    ]
  }
}
```

---

## HTTP status codes

| Status | When to use |
|--------|------------|
| `200 OK` | Successful GET, PATCH, DELETE |
| `201 Created` | Successful POST that creates a resource |
| `202 Accepted` | Request accepted for async processing |
| `204 No Content` | Successful DELETE with no response body |
| `400 Bad Request` | Invalid request body or parameters |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Authenticated but not authorized |
| `404 Not Found` | Resource does not exist |
| `409 Conflict` | Duplicate resource or state conflict |
| `422 Unprocessable Entity` | Validation errors |
| `429 Too Many Requests` | Rate limit exceeded |
| `500 Internal Server Error` | Unexpected server error |
| `503 Service Unavailable` | Downstream dependency unavailable |

---

## Error codes

Machine-readable `code` values (used in error responses):

| Code | Meaning |
|------|---------|
| `validation_failed` | Request body failed validation |
| `not_found` | Resource does not exist |
| `unauthorized` | Authentication required |
| `forbidden` | Insufficient permissions |
| `conflict` | Duplicate or conflicting state |
| `rate_limited` | Too many requests |
| `upstream_error` | External provider (Twilio, Mailgun, LLM) returned an error |
| `internal_error` | Unexpected server error |

---

## Authentication

All inter-service API calls use a shared secret header:

```
X-Whirlwind-Service-Token: <secret>
```

Secrets are provisioned per-caller via environment variables. Services validate the token on every request.

Public-facing product APIs (e.g., `mindblossom`) use JWT Bearer tokens:
```
Authorization: Bearer <jwt>
```

JWTs are issued by the product's auth system (Guardian). Scope claims determine what a token can access.

---

## Versioning

- All routes are versioned: `/v1/`, `/v2/`
- A breaking change (removed field, changed semantics, new required field) requires a new version
- Additive changes (new optional field, new endpoint) do not require a new version
- Old versions are deprecated with a sunset date in the response header:
  ```
  Sunset: Sat, 01 Jan 2027 00:00:00 GMT
  ```
- Old versions are supported for a minimum of 6 months after sunset announcement

---

## Pagination

All list endpoints are paginated. Default: 25 items per page, max 100.

```
GET /v1/messages?page=2&per_page=50
```

Response includes `meta.total`, `meta.page`, `meta.per_page`.

Cursor-based pagination is preferred for real-time feeds:
```
GET /v1/messages?after=<cursor>&limit=25
```

---

## Idempotency

POST endpoints that create resources accept an optional idempotency key:
```
Idempotency-Key: <uuid>
```

If the same key is sent twice within 24 hours, the second request returns the original response without creating a duplicate. This is critical for webhook handlers and job retry logic.

---

## Health endpoint

Every service exposes:
```
GET /health
```

Response:
```json
{
  "status": "healthy",
  "checks": {
    "database": "ok",
    "upstream_service": "ok"
  }
}
```

Returns `200` if healthy, `503` if any critical check fails.
