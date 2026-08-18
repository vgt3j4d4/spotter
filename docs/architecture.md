# Architecture

## Shape

Two deployables and a database.

```
┌──────────────┐   HTTPS/JSON   ┌──────────────────┐   JDBC   ┌────────────┐
│  Angular 20  │ ─────────────▶ │  Spring Boot 3   │ ───────▶ │ PostgreSQL │
│  (web/)      │ ◀───────────── │  (api/)          │ ◀─────── │            │
└──────────────┘                └──────────────────┘          └────────────┘
   served as                       stateless,
   static files                    JWT-authenticated
```

The frontend is a static bundle. The API is stateless — no server session, no sticky
routing — which is what makes horizontal scaling and the eventual ECS migration boring
rather than interesting.

## Modules in the API

Package-by-feature, not package-by-layer. `exercise`, `template`, `athlete`, `session`,
`measurement`, `sharing`, each holding its own controller, service, repository and
entities. A feature you delete is a directory you delete.

Inside a feature the layering is strict:

| Layer | Knows about | Never knows about |
|---|---|---|
| Controller | HTTP, DTOs, validation annotations | SQL, entities, other features' internals |
| Service | Domain rules, transactions, other services | HTTP, request objects, `HttpServletRequest` |
| Repository | JPA, queries | Anything above it |

Entities never leave the service layer. Controllers speak DTOs. This costs a mapping
step and buys the ability to change the schema without changing the API contract — which
is the entire point of having two of them.

## The auth asymmetry

This is the part worth understanding, and the part worth being able to defend.

**Trainers authenticate. Athletes do not.**

```
Trainer  ──▶  POST /auth/login  ──▶  JWT  ──▶  Authorization: Bearer …
Athlete  ──▶  https://spotter.app/s/<opaque-token>
```

A trainer has an account, a password, and a JWT carrying their identity. Every query in
the service layer is scoped to the authenticated trainer — an athlete belongs to exactly
one trainer, and there is no code path that reaches an athlete the caller does not own.

An athlete has no account at all. They have a URL containing a cryptographically random,
unguessable token. The token is the credential. Presenting it grants read access to one
athlete's routine and history, and nothing else.

**Why not give athletes accounts?** Because they would not use them. The person this is
built for is standing in a gym, mid-session, with sweaty hands. A password reset flow is
the point at which they stop opening the link and go back to asking the trainer. See
[ADR 0002](decisions/0002-athletes-have-no-accounts.md).

**What the token can reach** is a deliberately narrow surface — the routine, not the
person. Body measurements are trainer-only and never served to a token-authenticated
request. See [ADR 0003](decisions/0003-share-link-shows-the-routine-not-the-person.md).

Implementation: a separate `SecurityFilterChain` for `/s/**` and `/api/public/**` that
resolves the token, loads the athlete, and populates a read-only principal. Two chains
rather than one chain with branching, so the read-only path cannot accidentally inherit
a write authority.

## Request flow

```
Request
  └─▶ CorrelationIdFilter        assigns X-Request-Id, puts it in the MDC
      └─▶ SecurityFilterChain    JWT for /api/**, share token for /s/**
          └─▶ Controller         @Valid on the DTO, jakarta.validation
              └─▶ Service        @Transactional, ownership check, domain rules
                  └─▶ Repository
Response
  └─▶ RestControllerAdvice       maps exceptions to RFC 9457 Problem Details
      └─▶ CorrelationIdFilter    echoes X-Request-Id back
```

Every log line carries the correlation ID from the MDC, and the client gets the same ID
in the response header. When the trainer reports that "it broke," the ID from their
screenshot finds every line the request produced.

## Frontend

Standalone components, no NgModules. Feature-routed with lazy loading, so the athlete's
public view does not download the trainer's admin bundle.

- **State** — Angular signals for component state, services with `toSignal` for server
  data. No NgRx: this is CRUD, and the ceremony would exceed the complexity.
- **Forms** — reactive forms, typed. Validation mirrors the server's, and the server's
  is authoritative.
- **HTTP** — one interceptor attaches the JWT, one surfaces Problem Details as
  user-facing messages, one adds the correlation ID.
- **Offline** — a service worker caches the current session read-only, so a phone that
  loses signal in a basement gym still shows the routine.

## What is deliberately not here

No message queue, no cache layer, no microservices, no GraphQL. One trainer with a few
dozen athletes generates a workload PostgreSQL answers without noticing. Adding
infrastructure to demonstrate familiarity with infrastructure is how portfolio projects
stop being credible.
