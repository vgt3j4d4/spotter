# Contributing

This is a personal project, but it is developed as if it were not. The conventions
below exist so the history stays readable and the CI signal stays honest.

## Before you start

Every unit of work is a GitHub issue, and every issue belongs to a milestone. If the
work you are about to do is not an issue, file it first. See
[the roadmap](docs/roadmap.md) for the milestone order.

## Branches

Two long-lived branches:

| Branch | Holds | Written by |
|---|---|---|
| `main` | What is in production | Merges from `develop` only |
| `develop` | Work that is finished but not released | Merges from feature branches |

`develop` is the default branch. Branch off it, one branch per issue, and open the PR
back into it.

```
feat/23-set-logging-api
fix/61-rpe-null-on-bulk-log
chore/6-repo-hygiene
docs/adr-share-link-scope
```

Both long-lived branches are protected. No direct commits to either — everything arrives
by pull request.

## Releasing

Shipping to production is a PR from `develop` into `main`. It is a real review, not a
formality: it is the last point at which everything going live is visible in one diff.

```
release: S4 — scheduling and set logging
```

Tag `main` after the merge (`v0.4.0`), and let the deploy fire from the tag rather than
from the branch, so what is running is always identifiable.

`main` never receives a feature branch directly. The one exception is a hotfix: branch
from `main`, PR into `main`, then merge `main` back into `develop` immediately so the fix
is not lost on the next release.

## Commits

[Conventional Commits](https://www.conventionalcommits.org/). The type prefix drives
the changelog, so it is not decoration.

```
feat(api): log weight, reps and RPE per set
fix(web): keep focus in the set editor after autosave
test(api): assert a share token cannot reach measurement endpoints
```

Reference the issue in the body, and close it there:

```
Bulk-logging a session sent one request per set, which meant a
half-saved session when the trainer walked out of wifi range.

fixes #23
```

## Pull requests

Every change goes through a PR, including your own. The checklist:

- [ ] CI is green — build, tests, lint
- [ ] New behaviour has a test that fails without the change
- [ ] Public API changes are reflected in the springdoc annotations
- [ ] Schema changes ship as a Flyway migration, never as an edit to an applied one
- [ ] A decision with a real alternative got an ADR in `docs/decisions/`
- [ ] The issue is linked with `fixes #<id>`

Squash-merge. The PR title becomes the commit message, so write it as one.

## Tests

Run them before you open the PR — CI running them is not a substitute for you knowing.

```bash
./api/mvnw test                     # unit
./api/mvnw verify                   # + Testcontainers integration
npm --prefix web test               # Angular unit
npm --prefix web run e2e            # Playwright
```

Integration tests run against real PostgreSQL through Testcontainers. There is no H2
fallback and no in-memory substitute — a test that passes against a database you do not
deploy is not evidence of anything.

## Code style

Formatting is enforced in CI, not in review. Run the formatter and move on.

- **Java** — Spotless with google-java-format. Constructor injection only, no field `@Autowired`.
- **TypeScript** — Prettier and ESLint via the Angular defaults. Strict mode stays on.
- **Layering** — controller → service → repository. Controllers do not touch repositories,
  services do not know about HTTP, and entities do not cross the controller boundary.

## Documentation

Docs live next to the code and change with it. If a PR makes
[`docs/architecture.md`](docs/architecture.md) or
[`docs/data-model.md`](docs/data-model.md) wrong, fixing them is part of that PR.
