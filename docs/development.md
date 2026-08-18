# Development

## Prerequisites

| Tool | Version | Notes |
|---|---|---|
| JDK | 21 | Temurin recommended |
| Node | 22 LTS | |
| Docker | any recent | Required for PostgreSQL and Testcontainers |

## Running locally

```bash
docker compose up -d              # PostgreSQL on :5432
./api/mvnw spring-boot:run        # API on :8080, Flyway migrates on boot
npm --prefix web install
npm --prefix web start            # Angular dev server on :4200, proxied to :8080
```

Seed data — a demo trainer, a handful of exercises and a couple of athletes with history —
is loaded by the `local` Spring profile, which is active by default in
`compose.yaml`. The app is usable the moment it boots; nothing has to be clicked into
existence first.

## Configuration

Local defaults live in `api/src/main/resources/application-local.yaml` and are committed,
because they point at a throwaway container. Anything secret is read from the environment
and has no committed default:

| Variable | Purpose |
|---|---|
| `SPOTTER_JWT_SECRET` | Signing key for trainer tokens |
| `SPOTTER_DB_URL` / `_USER` / `_PASSWORD` | Overrides the compose defaults |

There is no `.env` in git. `.env.example` documents the shape.

## Tests

```bash
./api/mvnw test          # unit — fast, no Docker
./api/mvnw verify        # + integration via Testcontainers
npm --prefix web test    # Angular unit
npm --prefix web run e2e # Playwright, against a running stack
```

**Integration tests run against real PostgreSQL.** Testcontainers starts one, Flyway
migrates it, the test runs, the container dies. There is no H2 profile and there will not
be one: H2 does not have `DISTINCT ON`, does not enforce the same constraints, and a suite
that is green against a database you do not deploy tells you nothing about the one you do.

The container is reused across the suite via `.testcontainers.properties` — starting
PostgreSQL once per class is the difference between a suite you run and a suite you skip.

## Database changes

Every schema change is a new Flyway migration in
`api/src/main/resources/db/migration`, named `V<n>__snake_case_description.sql`.

Never edit an applied migration. Flyway checksums them, and editing one breaks every
environment that already ran it — including your teammate's laptop and production.
Corrections go forward as a new migration.

## Debugging

Every request carries a correlation ID. It is in the `X-Request-Id` response header, in
the MDC of every log line the request produced, and in the `requestId` field of any
Problem Details error body.

```bash
docker compose logs api | grep 3f9c1a2e
```

That is the entire debugging workflow for "a trainer says it broke and sent a screenshot."

## Conventions

Formatting is enforced in CI. Run it locally first:

```bash
./api/mvnw spotless:apply
npm --prefix web run lint -- --fix
```

Constructor injection only in Spring — no field `@Autowired`. It keeps the class testable
without a container and makes an over-large dependency list visibly uncomfortable, which
is the correct feedback.
