# API

Base path `/api/v1`. JSON in, JSON out. The generated OpenAPI page is at `/swagger-ui.html`
and is the authoritative reference — this document covers the conventions the generated
page cannot express.

## Authentication

Two entirely separate paths.

**Trainers** exchange credentials for a JWT and send it as a bearer token.

```http
POST /api/v1/auth/login
{ "email": "…", "password": "…" }

→ 200 { "token": "eyJ…", "expiresAt": "2026-08-19T10:00:00Z" }
```

```http
GET /api/v1/athletes
Authorization: Bearer eyJ…
```

**Athletes** present an opaque token in the path. No header, no login, no account.

```http
GET /api/v1/public/s/{token}/session
```

The two paths are served by separate security filter chains. The public chain resolves
the token to a read-only principal and cannot reach a write endpoint — not by convention,
but because those endpoints are not in its chain.

## Errors — RFC 9457

Every error response is `application/problem+json`. No bare strings, no `{"error": "…"}`,
no HTML error pages.

```json
{
  "type": "https://spotter.app/problems/validation-failed",
  "title": "Validation failed",
  "status": 400,
  "detail": "2 fields are invalid",
  "instance": "/api/v1/athletes/12/measurements",
  "requestId": "3f9c1a2e-…",
  "errors": [
    { "field": "values[0].value", "message": "must be greater than 0" },
    { "field": "takenOn",         "message": "must be a past or present date" }
  ]
}
```

`requestId` matches the `X-Request-Id` response header and the correlation ID in every
log line the request produced. It is the reason a bug report can be resolved from a
screenshot.

| Status | When |
|---|---|
| 400 | Validation failed, or a malformed request body |
| 401 | Missing, expired or invalid credentials |
| 403 | Authenticated, but the resource belongs to another trainer |
| 404 | No such resource — also returned instead of 403 where existence itself is sensitive |
| 409 | Conflict, e.g. two measurement sessions for one athlete on one date |
| 422 | Well-formed and valid, but rejected by a domain rule |

**403 versus 404.** Requesting another trainer's athlete returns 404, not 403. A 403
confirms the row exists, which is a small enumeration oracle. Within your own data, 403
is used honestly.

## Validation

`jakarta.validation` on request DTOs, enforced by `@Valid` at the controller boundary.
Constraint violations are collected and returned together — a form that fails one field
at a time is a form nobody finishes.

The server's validation is authoritative. The Angular reactive forms mirror it for
responsiveness, and that mirror is a convenience, never a substitute.

## Conventions

- **Pagination** — `?page=0&size=20&sort=takenOn,desc`. Responses carry `content`,
  `page`, `size`, `totalElements`, `totalPages`.
- **Dates** — `taken_on` and `session_date` are `LocalDate`, serialised `YYYY-MM-DD`, no
  timezone. A workout happened on a day, not at an instant, and attaching a timezone to
  it creates a bug at every UTC boundary.
- **Timestamps** — audit fields are `Instant`, serialised ISO-8601 with `Z`.
- **Units** — every numeric body measurement and weight crosses the wire in canonical
  units (kg, cm). Conversion for display happens in the client.
- **Partial updates** — `PATCH` with only the changed fields. `PUT` is not used; nobody
  sends a complete representation correctly.
- **Idempotency** — `PUT` and `DELETE` are idempotent. Deleting an already-deleted
  resource returns 204, not 404.

## Endpoint groups

| Group | Path | Auth |
|---|---|---|
| Auth | `/auth/**` | none |
| Exercises | `/exercises` | trainer |
| Templates | `/templates` | trainer |
| Athletes | `/athletes` | trainer |
| Scheduled sessions | `/athletes/{id}/sessions` | trainer |
| Logged sets | `/sessions/{id}/sets` | trainer |
| Measurements | `/athletes/{id}/measurements` | trainer **only** |
| Share tokens | `/athletes/{id}/share` | trainer |
| Public athlete view | `/public/s/{token}/**` | share token |

The measurements row is load-bearing. There is no public measurement endpoint, and
[an integration test asserts it](decisions/0003-share-link-shows-the-routine-not-the-person.md).
