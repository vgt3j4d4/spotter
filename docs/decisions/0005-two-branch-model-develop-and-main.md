# 0005 — Two-branch model: `develop` and `main`

**Status:** Accepted · 2026-08-18

## Context

Work needs somewhere to accumulate before it is in front of the trainer. Merging a
half-finished milestone straight onto the branch that deploys to production means either
deploying it or blocking the deploy — both bad.

The project also has a real user from early on. "It is on the internet" and "the trainer
should be using it" need to be distinguishable states.

## Decision

Two long-lived branches.

- **`main`** is production. What is on it is what is deployed. It only ever receives
  merges from `develop`, plus hotfixes.
- **`develop`** is the default branch and where work integrates. Feature branches cut from
  it and PR back into it.

A release is a pull request from `develop` into `main`, reviewed as a real change, tagged
after merge. Deploys fire from the tag, not the branch, so the running version is always
identifiable.

A hotfix branches from `main`, PRs into `main`, and is merged back down into `develop`
immediately — otherwise the next release silently reverts it.

## Alternatives rejected

**Trunk-based development on `main` alone.** Genuinely the better default for a team
shipping several times a day behind feature flags. Rejected here because the flag
infrastructure that makes it safe does not exist, this is roughly ten hours a week rather
than a full-time team, and a half-built feature would sit exposed on the production branch
for days. The thing that makes trunk-based work is continuous deployment; without it, it
is just an unprotected production branch.

**Full git-flow** — `develop`, `main`, plus `release/*`, `hotfix/*` and `feature/*`
prefixes. The release branches solve stabilising a release while new work continues. With
one developer, nothing continues during stabilisation, so those branches would be
ceremony with no function.

**A branch per environment** (`dev`, `staging`, `prod`). Three branches to keep in sync
and a standing invitation for them to diverge. Two environments do not need three
branches.

## Consequences

**Good.** `main` is always deployable and always reflects what the trainer is using.
Work in progress has somewhere to live. The release PR is a single diff of everything
about to go live, which is the last useful moment to catch something.

**Bad.** Every change is two merges instead of one, and `develop` can drift ahead of
`main` far enough that a release diff stops being reviewable. The mitigation is releasing
per milestone rather than per quarter — if the release PR is too big to read, it waited
too long.

**Revisit this** if the project ever gets feature flags and a real CD pipeline. At that
point trunk-based becomes the better answer and this record should be superseded rather
than quietly ignored.
