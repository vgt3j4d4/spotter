# 0003 — The share link shows the routine, not the person

**Status:** Accepted · 2026-08-18

## Context

[ADR 0002](0002-athletes-have-no-accounts.md) established that athletes reach their data
through an unguessable URL rather than an account. That URL is forwardable by design —
it gets pasted into WhatsApp, and WhatsApp messages get forwarded.

Body measurements ([S5](../roadmap.md)) were added to the schema after the sharing design
was drafted: bodyweight, body-fat percentage, and a dozen circumferences per athlete over
time. The obvious next step was to surface them in the athlete's view, since progress over
time is genuinely motivating and the athlete is the person it is about.

That would change what the link *is*. Before: a workout plan. After: a named person's body
composition, on a URL that anyone holding can forward to anyone else, with no expiry and
no audit trail.

## Decision

The share link exposes the routine and the training history. It does not expose body
measurements.

Measurements are trainer-only. There is no public measurement endpoint, and the exclusion
is enforced by the security filter chain rather than by the UI declining to render them.
An integration test asserts that a share token receives 404 from every measurement route.

The link stays athlete-scoped and durable — one token per athlete, opening on the next
scheduled session — rather than one token per session.

## Alternatives rejected

**Measurements behind a per-athlete opt-in flag.** Better than always-on, but it makes a
consent decision into a checkbox on a settings screen, where the default wins and nobody
reads it. If the trainer asks for this later it can be built deliberately, with the
consent conversation designed in.

**Rendering measurements but disabling edits.** Confuses "read-only" with "not exposed."
The data is still on the wire.

**A token per scheduled session.** Tighter — a forwarded link leaks one workout and then
expires. Rejected because it means the trainer performs a send-a-link chore before every
session, and friction is what kills a tool like this in week three. The durable link that
always opens on the next session behaves like "today's routine" without anyone resending
it.

**Doing nothing and shipping measurements to the link.** This was the original plan. It
was reversed when the actual use case was stated plainly: the athlete wants the routine
for the day, not a profile page.

## Consequences

**Good.** The link carries what the athlete asked for and nothing more sensitive than
their own workout. The health-data question is closed rather than deferred, and the answer
is enforced in the filter chain where it cannot be undone by a UI change.

**Bad.** When the trainer takes measurements, showing them to the athlete means turning
the phone around. Both are in the same room at that moment, so the cost is close to zero —
but it is a real limitation and it will feel like a missing feature to anyone who has not
read this file.

**Residual risk, accepted.** The routine link is still forwardable and still carries the
athlete's name. Revocation and rotation ([#29](https://github.com/vgt3j4d4/spotter/issues/29))
are the mitigation. This is a training log, not a medical record, and the trade is
deliberate.
