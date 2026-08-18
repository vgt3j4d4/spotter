# Spotter

Workout planning and session history for personal trainers.

A trainer keeps a catalogue of exercises, builds reusable workout templates, schedules
sessions per athlete per day, and logs what was actually lifted — weight, reps, RPE —
from a phone on the gym floor. Athletes get a link. No account, no password, no app.

**Status: in development.** Nothing is deployed yet. See [the roadmap](docs/roadmap.md)
for what is being built and in what order.

---

## Why this exists

It is built for one real trainer with real clients, which is the only reason it makes
the trade-offs it does. Every design decision in [`docs/decisions/`](docs/decisions/)
traces back to something that happens on a gym floor: the trainer has one hand free,
the athlete will not install anything, and the useful number mid-set is what they lifted
last Tuesday.

## Stack

| Layer | Choice |
|---|---|
| Frontend | Angular 20, Angular Material, TypeScript |
| Charts | Chart.js via `ng2-charts` |
| Dates | `date-fns` |
| API | Spring Boot 3, Java 21 |
| Persistence | PostgreSQL 16, Spring Data JPA, Flyway |
| Auth | Spring Security, JWT for trainers; signed tokens for athlete links |
| Validation | `jakarta.validation` |
| Errors | RFC 9457 Problem Details |
| Logging | Logback with MDC correlation IDs |
| API docs | springdoc OpenAPI |
| Testing | JUnit 5, Mockito, Testcontainers, Playwright |
| Local dev | Docker Compose |
| CI | GitHub Actions |
| Hosting | Fly.io, migrating to AWS (ECS Fargate + RDS) post-launch |

## Layout

```
api/     Spring Boot service
web/     Angular application
docs/    Architecture, data model, API contract, decision records
```

## Getting started

Requires Docker, JDK 21 and Node 22.

```bash
docker compose up -d      # PostgreSQL
./api/mvnw spring-boot:run
npm --prefix web start
```

Full setup, test commands and troubleshooting are in
[`docs/development.md`](docs/development.md).

## Documentation

| Document | What it covers |
|---|---|
| [Architecture](docs/architecture.md) | Modules, request flow, the trainer/athlete auth asymmetry |
| [Data model](docs/data-model.md) | Tables, relationships, and why the schema is shaped this way |
| [API](docs/api.md) | Endpoints, error contract, authentication |
| [Development](docs/development.md) | Local setup, running tests, conventions |
| [Roadmap](docs/roadmap.md) | Milestones and what ships when |
| [Decisions](docs/decisions/) | Architecture decision records |

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Licence

[MIT](LICENSE) © 2026 Gonzalo Tejada.

The licence covers the source code in this repository. It does not cover athlete data,
measurements, or any training content entered by users of a deployed instance — that
belongs to whoever runs it and the people it describes.
