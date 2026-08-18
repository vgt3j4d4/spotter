# Testing

## When tests get written

**With the feature, in the same pull request.** Not in a milestone at the end.

`S8 · Testing & polish` is not where the tests are written. It is where the things that
can only be done once the app is complete happen: end-to-end journeys across finished
flows, the accessibility audit, the coverage gate, mutation testing. If S8 is the first
time a service gets a unit test, the milestone has failed and so has everything before it.

The PR checklist says it in one line: *new behaviour has a test that fails without this
change.* A test written after the code has already been seen working tests what the code
does. A test written alongside it tests what the code should do. Those are different, and
only one of them catches the regression six weeks later.

## Levels

Four, each answering a different question. The cost goes up and the count goes down.

| Level | Question | Tool | Speed |
|---|---|---|---|
| Unit | Is this logic correct? | JUnit 5, no Spring context | milliseconds |
| Slice | Does this layer behave at its boundary? | `@WebMvcTest`, `@DataJpaTest` | fast |
| Integration | Do the layers work together against a real database? | `@SpringBootTest` + Testcontainers | seconds |
| End-to-end | Can a person actually do the thing? | Playwright | slow |

### Unit — the bulk of it

Pure logic, no Spring, no database, no mocks where a real object will do. Unit conversion,
validation rules, the shape of a session projection, date arithmetic.

If a class needs the Spring context to be tested, that is a design signal before it is a
testing problem — the logic is tangled with the framework and should be extracted.

### Slice — the boundaries

`@WebMvcTest` for controllers: does a bad payload produce a 400 with the right Problem
Details body, does an unauthenticated request produce a 401, does another trainer's athlete
produce a 404. The service is mocked here, because the question is about the HTTP boundary.

`@DataJpaTest` against Testcontainers for repositories: does the query return what it
claims. Never mocked — a mocked repository test asserts that the mock was configured, which
is not information.

### Integration — the flows that cross everything

Full context, real Postgres, real Flyway migrations. Reserved for behaviour that only
exists when the layers are assembled: authentication, ownership scoping, share-token
access, bulk set logging in one transaction.

**Real PostgreSQL, always.** No H2, no in-memory substitute. The schema depends on
`DISTINCT ON`, on real constraint behaviour, and on types H2 approximates rather than
implements. A suite green against a database you do not deploy is not evidence about the
one you do.

### End-to-end — a handful, no more

Playwright, over real user journeys, on the deployed build. These are the tests that break
for reasons unrelated to the change that broke them, so there are few of them and each one
earns its place:

1. Trainer logs in, creates an exercise, builds a template
2. Trainer schedules a session and logs sets against it
3. Trainer records a measurement session and sees it on the chart
4. Athlete opens a share link and sees a read-only routine

E2E is not a regression suite. Anything that can be caught a level down is caught a level
down.

## What must be tested in this application

Coverage percentage is a weak signal. This list is the strong one — these are the
assertions that must exist, and a PR that weakens one of them is wrong regardless of what
the coverage number says.

| Assertion | Why | Level |
|---|---|---|
| A share token cannot reach any measurement endpoint | The promise made in [ADR 0003](decisions/0003-share-link-shows-the-routine-not-the-person.md). Health data, on a forwardable URL | Integration |
| A share token cannot reach any write endpoint | The link is read-only by construction, and that must be proven, not assumed | Integration |
| Trainer A cannot read or write Trainer B's athletes | Every service method is scoped by owner; one missed scope is a data breach | Integration |
| Another trainer's resource returns 404, not 403 | 403 confirms existence. The distinction is deliberate and easy to regress | Slice |
| Unit conversion round-trips | Convert to imperial, convert back, land on the value you started with. Property-based | Unit |
| The last-session query returns the correct prior session | The `DISTINCT ON` is the core of the product and silently returning the wrong session is worse than an error | Slice (`@DataJpaTest`) |
| Validation bounds reject impossible measurements | A fat-fingered `1750 cm` ruins every chart it appears in | Slice |
| A revoked share token fails closed | Fails open means revocation does nothing and nobody notices | Integration |

Each of these ships in the PR that builds the feature it protects.

## What not to test

- **Getters, setters, and DTO mapping.** Coverage without information.
- **The framework.** Spring's `@Valid` works. Testing that it works tests Spring.
- **Mocked repositories in a service test *and* the same behaviour again at the repository
  level with a mocked database.** Pick a level. Two mocks that agree with each other prove
  nothing.
- **Implementation detail.** A test that breaks when a method is renamed but nothing
  observable changed is a test that will be deleted the first time it is inconvenient.

The failure mode to watch for is over-mocking: a suite where every collaborator is a mock
passes cheerfully while the assembled application is broken, because nothing in it ever
ran together.

## Tooling

| Purpose | Tool |
|---|---|
| Java unit and slice | JUnit 5, AssertJ |
| Mocking collaborators | Mockito — for things the test does not own |
| Property-based | jqwik |
| Real database | Testcontainers PostgreSQL |
| HTTP assertions | MockMvc, RestAssured for full-context tests |
| Angular unit | Jest with Testing Library |
| Angular HTTP | `HttpTestingController` — no real network in a unit test |
| E2E | Playwright |

AssertJ over JUnit's built-in assertions: `assertThat(sets).extracting("reps").containsExactly(8, 8, 6)`
fails with a message that says what went wrong, which is the entire value of an assertion
library.

Testing Library over Angular's default harness where they overlap. Querying by role and
label rather than by CSS class means the test breaks when the user-visible behaviour breaks,
not when a class is renamed — and it makes accessibility failures show up as test failures.

## CI

Every push runs unit, slice and integration. E2E runs on pull requests into `develop` and
`main`, against a built app.

Testcontainers reuse is enabled — starting PostgreSQL once per suite instead of once per
class is the difference between a suite that gets run locally and one that gets skipped.

## Coverage and mutation testing

Coverage is a gate against *nothing*, not a target. Set it low enough to catch an untested
feature and no higher; a high threshold produces tests written for the threshold, which are
worse than no tests because they look like protection.

**Mutation testing (PIT) is the real measure.** It changes the code — flips a conditional,
swaps an operator, removes a call — and asks whether any test noticed. A suite at 90%
coverage that survives most mutations is a suite that executes code without asserting on
it. Running PIT over the service and domain packages is what turns "we have tests" into
evidence.

It is slow, so it runs on a schedule and on release PRs, not on every push.
