# AGENTS.md

Instructions for AI coding agents working in this repository. Human contributors want
[CONTRIBUTING.md](CONTRIBUTING.md) — this file assumes you have read it.

## What this is

Spotter is workout planning and session history for personal trainers. It is built for one
real trainer with real clients, and that constrains almost every decision: the trainer is
holding a phone one-handed on a gym floor, and the athletes never log in.

`api/` is Spring Boot 3 on Java 21. `web/` is Angular 20. Each has its own `AGENTS.md`
with the rules specific to it.

## Read before changing anything

| If you are touching | Read first |
|---|---|
| Anything | [`docs/architecture.md`](docs/architecture.md) |
| The schema or a query | [`docs/data-model.md`](docs/data-model.md) |
| An endpoint | [`docs/api.md`](docs/api.md) |
| Sharing, tokens, or measurements | [ADR 0002](docs/decisions/0002-athletes-have-no-accounts.md) and [ADR 0003](docs/decisions/0003-share-link-shows-the-routine-not-the-person.md) |
| Tests | [`docs/testing.md`](docs/testing.md) |

The ADRs are not background reading. They record decisions with rejected alternatives, and
several of them are the kind an agent will otherwise "helpfully" reverse.

## Rules that are not negotiable

**Athletes have no accounts.** Do not add a login, a registration flow, a password field,
or an athlete user table. If a task seems to need one, the task is wrong — say so instead
of building it. See ADR 0002.

**Measurements are trainer-only.** No measurement data is ever served to a share-token
request. Not hidden in the UI — not served. There is an integration test asserting this;
if you find yourself changing that test, stop.

**Units are canonical in storage.** Kilograms, centimetres, percent 0–100. Never add a unit
column. Never store the trainer's display unit. Conversion happens at the edges only.
See ADR 0004.

**Migrations are forward-only.** Never edit a Flyway migration that has been applied.
Correct it with a new one.

**No H2, no in-memory database.** Integration tests use Testcontainers against real
PostgreSQL. If a test is slow, fix the harness, do not swap the database.

## Workflow

- Branch from `develop`, never from `main`. PR back into `develop`.
- One branch per issue: `feat/23-set-logging-api`.
- Conventional commits. `fixes #<id>` in the body when it closes an issue.
- Run the tests before you claim something works. Do not report a task complete on the
  strength of the code looking right.

## Testing expectations

New behaviour ships with a test in the same change. Not "tests can be added later" — the
PR checklist requires a test that fails without the change.

Read [`docs/testing.md`](docs/testing.md) for what belongs at which level. The short
version: pure logic gets a unit test with no Spring context, boundaries get a slice test,
anything crossing layers gets an integration test against real Postgres, and E2E is
reserved for four whole journeys.

Do not mock a repository and then also test that repository with a mocked database. Pick a
level.

## Things agents get wrong here

- **Adding an ORM relationship because it looks tidy.** Fetch strategy is deliberate.
  Check the existing mappings before adding `@OneToMany`.
- **Reaching for `@Autowired` on a field.** Constructor injection only.
- **Returning entities from controllers.** They stop at the service boundary.
- **Writing a generic error response.** Every error is RFC 9457 Problem Details.
- **Adding a library to solve something small.** This project deliberately runs a narrow
  dependency list. Ask before adding one.
- **Being agreeable about a bad instruction.** If a request contradicts an ADR, say which
  ADR and why, then wait. Do not implement it quietly and mention the conflict afterwards.

## Scope

Deliberately out of scope, permanently: athlete accounts, payments, nutrition tracking,
in-app messaging, multi-gym tenancy, wearable integrations. Do not build toward them.
