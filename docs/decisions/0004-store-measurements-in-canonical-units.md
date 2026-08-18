# 0004 — Store measurements in canonical units

**Status:** Accepted · 2026-08-18

## Context

Trainers work in metric or imperial depending on where they trained and who they trained
with, and the same trainer will happily say "sixty kilos" and "a fifteen-inch arm" in the
same sentence. The app has to display whichever the user expects.

The tempting schema stores the number the user typed alongside the unit they typed it in:
`value NUMERIC, unit VARCHAR`. It requires no conversion, no rounding, and no thought.

It also produces a table where `180` might be pounds or kilograms, and where every query
that compares, charts, or aggregates has to convert first — or silently does not.

## Decision

All measurements and weights are stored in canonical units: **mass in kilograms, length in
centimetres, percentages as 0–100.** There is no unit column anywhere in the schema.

The trainer's `unit_system` preference (`METRIC` or `IMPERIAL`) lives on their profile and
is applied at the edges only: converted for display on the way out, converted back on the
way in before the value reaches the API. The wire format is canonical in both directions.

Rounding happens for display and never before persistence. Every displayed value carries
its unit label — always, including in tables where it feels repetitive.

## Alternatives rejected

**Value plus unit column.** Cheap to write, expensive forever after. Every read is a
conversion, every aggregate is a bug waiting for the one row in the other unit, and a row
whose unit was recorded wrong is unrecoverable — nobody remembers what a number from
eighteen months ago meant.

**Store in the trainer's preferred unit and convert on preference change.** A destructive
migration of live data every time someone toggles a setting. If it fails halfway, the table
is mixed-unit with no way to tell which rows were converted.

**Imperial as canonical.** Same properties, worse arithmetic, and the seeded site list and
plate weights are metric anyway.

## Consequences

**Good.** Comparison, charting and aggregation are arithmetic on comparable numbers. The
unit preference is a rendering concern and changing it is instantaneous and reversible. A
value read straight out of the database is unambiguous.

**Bad.** Round-tripping through imperial introduces display drift — a trainer who types
`15.0` inches sees `15.0` back, but the stored `38.1` cm redisplayed after a preference
toggle may render as `15.0` only because of rounding, and a value entered as `38.2` cm
will show as the same `15.0`. Two distinct stored values can present identically. This is
inherent to unit conversion, not to this decision.

**Test obligation.** Conversion is unit-tested in both directions with a round-trip
property check: convert to imperial, convert back, land on the value you started with
within the display tolerance. That test is the thing standing between this decision and a
class of bug that is invisible until it is embarrassing.
