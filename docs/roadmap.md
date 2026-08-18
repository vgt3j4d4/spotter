# Roadmap

Nine milestones, 49 issues. The
[GitHub milestones](https://github.com/vgt3j4d4/spotter/milestones) are the live version;
this page explains the ordering.

| # | Milestone | Issues | Ships |
|---|---|---|---|
| S1 | Foundation & deploy | 7 | An empty app, live on the internet |
| S2 | Domain & API | 8 | Schema, layering, auth, error contract, logging |
| S3 | Catalogue & templates | 4 | Exercises and reusable workouts |
| S4 | Scheduling & logging | 8 | The core loop — plan a session, log what was lifted |
| S5 | Body measurements | 6 | Measurements over time, with progress charts |
| S6 | Athlete sharing | 3 | The link the athlete opens |
| S7 | PWA & mobile | 3 | Installable, one-handed, survives bad signal |
| S8 | Testing & polish | 7 | Test suite, accessibility, demo data, README |
| S9 | AWS migration | 3 | Post-launch — ECS Fargate and RDS |

## Why this order

**Deploy first, before there is anything to deploy.** S1 ends with a hello-world page
live in production and CI green on every push. Standing up a deploy pipeline against an
empty app takes an afternoon; standing one up against a finished app takes a week, and
you discover it the week you wanted to be done.

**Contracts before features.** S2 builds no user-facing behaviour at all — it establishes
the layering, the validation approach, the RFC 9457 error shape and correlation-ID
logging. Every feature after it inherits those for free. Retrofitting an error contract
across fifteen endpoints is the kind of work that never gets done.

**Catalogue before scheduling.** You cannot schedule a workout made of exercises that do
not exist yet. S3 is small and unglamorous and everything downstream depends on it.

**S4 is the product.** Plan a session, log the sets, see what they lifted last time. If
the project shipped after S4 and nothing else, it would still be useful to the trainer
on Monday. Everything before it is scaffolding and everything after it is improvement.

**Measurements before sharing, deliberately.** S5 lands first so that S6 is designed with
the full data set in view. Building the share link while measurements exist forces the
question "what does this link expose?" to be answered explicitly rather than inherited by
accident — which is exactly how that decision got made. See
[ADR 0003](decisions/0003-share-link-shows-the-routine-not-the-person.md).

**S7 before S8.** The PWA work changes how the app boots and caches, which invalidates
E2E tests written against the non-PWA build. Writing the suite after means writing it
once.

**S9 is after launch, not before.** Fly.io gets the app in front of a real user in a
weekend. AWS is the migration that demonstrates ECS, RDS and a real infrastructure story —
worth doing, worth doing *second*. Nobody is helped by an app that is architecturally
ready for scale and not yet usable by the one person who asked for it.

## Definition of done

A milestone is done when it is merged to `develop`, green in CI, released to `main`,
deployed, and reachable by the trainer without an explanation.

Merged to `develop` is not done. Done is the trainer opening it.
