# 0002 — Athletes have no accounts

**Status:** Accepted · 2026-08-18

## Context

The trainer needs to give each athlete their routine. The athlete needs to read it,
usually on a phone, usually while already in the gym.

The default answer is user accounts: the athlete registers, logs in, and sees their plan.
That brings registration, email verification, password reset, session management, and a
support burden that lands on the trainer.

The people this is built for are not going to do that. They are standing in a gym with one
hand free. A forgotten password at that moment does not lead to a password reset — it
leads to asking the trainer, and then to not opening the app again.

## Decision

Athletes do not have accounts. Each athlete has an unguessable, cryptographically random
token; the trainer sends them a URL containing it. Holding the URL grants read-only access
to that one athlete's routine and history.

Trainers do authenticate, with a password and a JWT. The asymmetry is the point: the
person doing administration gets an account, the person doing a workout gets a link.

## Alternatives rejected

**Full athlete accounts.** Correct in a product with a growth team and a support inbox.
Here it adds five flows nobody asked for and a reason to stop using the app.

**Magic-link login.** Still an email round-trip, still an inbox, still a moment where it
does not work. All the friction of an account for most of the benefit of none.

**A shared password per trainer.** Every athlete sees every other athlete's data. Not
viable.

**A short numeric code.** Guessable. A four-digit code over a few dozen athletes is not a
credential.

## Consequences

**Good.** Zero onboarding. The athlete taps a link and reads their workout. Nothing to
install, nothing to remember, no support burden on the trainer.

**Bad.** The URL *is* the credential, so anyone it is forwarded to has the same access.
This is mitigated but not eliminated: tokens are revocable and rotatable
([#29](https://github.com/vgt3j4d4/spotter/issues/29)), the view is strictly read-only and
enforced server-side, the public pages carry `noindex`, and what sits behind the link is
deliberately narrow — see [ADR 0003](0003-share-link-shows-the-routine-not-the-person.md).

**Also bad.** There is no athlete identity, so features that would need one — the athlete
logging their own sets, commenting, tracking anything themselves — are off the table
without revisiting this record. That is an accepted limit, not an oversight.
