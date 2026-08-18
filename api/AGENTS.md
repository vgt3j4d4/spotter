# AGENTS.md — `api/`

Spring Boot 3, Java 21, PostgreSQL. Read the [root AGENTS.md](../AGENTS.md) first.

> The service does not exist yet — it is scaffolded in
> [#1](https://github.com/vgt3j4d4/spotter/issues/1). These are the rules it is built to.

## Commands

```bash
./mvnw spring-boot:run       # requires docker compose up -d from the repo root
./mvnw test                  # unit and slice
./mvnw verify                # + integration via Testcontainers
./mvnw spotless:apply        # formatting, enforced in CI
```

## Structure

Package by feature, not by layer:

```
com.spotter.exercise/     ExerciseController, ExerciseService, ExerciseRepository, …
com.spotter.athlete/
com.spotter.session/
com.spotter.measurement/
com.spotter.sharing/
com.spotter.common/       error handling, correlation IDs, config
```

A feature you delete should be a directory you delete. Do not create `controllers/`,
`services/`, `dtos/` packages.

## Layering

| Layer | May use | Must not use |
|---|---|---|
| Controller | DTOs, `@Valid`, services | Repositories, entities, SQL |
| Service | Entities, repositories, other services | `HttpServletRequest`, HTTP status, `ResponseEntity` |
| Repository | JPA, queries | Anything above it |

**Entities never leave the service layer.** Controllers accept and return DTOs. Yes, this
costs a mapping step. It is what lets the schema change without changing the API contract.

**Constructor injection only.** No field `@Autowired`. A constructor with eight parameters
is uncomfortable to look at, and that discomfort is correct feedback about the class.

## Persistence

- **Flyway, forward-only.** Never edit an applied migration.
- **`NUMERIC`, never `DOUBLE`,** for weights and measurements. Float error turns 62.5 into
  62.499999999999996 exactly where a trainer will see it.
- **Check the fetch strategy.** Default to `LAZY` on every association. An `EAGER` mapping
  added casually is an N+1 discovered in production.
- **Watch for N+1.** The last-session lookup is deliberately one `DISTINCT ON` query — see
  [`docs/data-model.md`](../docs/data-model.md). Do not replace it with a loop, however
  much more readable the loop looks.

## Validation and errors

- `jakarta.validation` on request DTOs, `@Valid` at the controller boundary. Not `javax` —
  Spring Boot 3 moved namespaces.
- Collect all violations and return them together. A form that fails one field at a time is
  a form nobody finishes.
- Every error response is RFC 9457 Problem Details via the `@RestControllerAdvice`. Never
  return a bare string or an ad-hoc `{"error": "..."}` shape.
- Another trainer's resource returns **404, not 403** — a 403 confirms it exists. Within
  the caller's own data, use 403 honestly.

## Security

Two filter chains, deliberately separate:

- `/api/v1/**` — JWT, trainer identity, full access to that trainer's data
- `/api/v1/public/s/{token}/**` — share token, read-only principal

They are separate chains rather than one chain with branching, so the read-only path
**cannot** inherit a write authority by accident.

Every service method that touches athlete-owned data scopes by the authenticated trainer.
This is enforced in the service, not the controller — a new controller must not be able to
bypass it by forgetting a check.

**No measurement data is reachable from the public chain.** Ever.

## Logging

pino's equivalent here is Logback with MDC. The correlation ID is set by a filter, lives in
the MDC, appears in every log line, and is echoed as `X-Request-Id`.

Do not log request bodies. They contain athlete names and body measurements.

## Testing

See [`docs/testing.md`](../docs/testing.md). In this module:

- Pure logic → JUnit 5, **no Spring context**. If a class needs the context to be tested,
  extract the logic.
- Controllers → `@WebMvcTest`, service mocked. Assert status codes and Problem Details
  bodies.
- Repositories → `@DataJpaTest` against Testcontainers. Never mocked.
- Cross-layer behaviour → `@SpringBootTest` + Testcontainers.
- AssertJ for assertions. Mockito only for collaborators the test does not own.
