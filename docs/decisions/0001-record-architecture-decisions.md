# 0001 — Record architecture decisions

**Status:** Accepted · 2026-08-18

## Context

This project makes a number of choices that look arbitrary from the outside and are not:
athletes without accounts, measurements in a reference-table schema, PostgreSQL over
MySQL, no state-management library. In six months the reasoning will have evaporated, and
the most likely outcome is that a future version of me reverses a decision without knowing
what it was protecting against.

## Decision

Record each such decision as a short numbered file in `docs/decisions/`, following MADR.
A record states the context, the decision, the alternatives that were rejected, and the
consequences — including the bad ones.

Records are immutable once accepted. A reversal is a new record that supersedes the old
one; the old file stays in place.

Only decisions with a genuine alternative get a record. "Use Flyway for migrations" is not
a decision, it is the obvious choice.

## Consequences

Writing one costs about fifteen minutes. In exchange, "why is it like this?" has an
answer that does not depend on anyone's memory, and the answer is in the repository rather
than in a chat log.

The failure mode is records that describe what the code does instead of why. A record that
does not name a rejected alternative is not doing its job.
