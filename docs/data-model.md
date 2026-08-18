# Data model

## Overview

```
Trainer ──▶ Exercise           (name, muscle group, equipment, notes, video URL)
        ──▶ WorkoutTemplate    "Push Day A"
             └─ TemplateItem   (exercise, target sets/reps/rest, order)
        ──▶ Athlete            (name, notes, share_token)
             ├─ ScheduledSession   (date, template)
             │    └─ LoggedSet     (exercise, set no., weight, reps, RPE)
             └─ Measurement        (taken_on, notes)
                  └─ MeasurementValue (site, value)
```

Everything hangs off `trainer`. There is no row in this schema that is not owned,
transitively, by exactly one trainer — which makes the ownership check in the service
layer a single join rather than a per-feature judgement call.

## Plan versus record

The most important distinction in the schema, and the one that is easy to get wrong.

- **`TemplateItem`** is the *plan*: "three sets of eight at around 60 kg."
- **`LoggedSet`** is the *record*: "set two, 62.5 kg, seven reps, RPE 8."

They are separate tables because they answer different questions and change at different
times. A template is edited between sessions; a logged set is written during one and then
never changes. Collapsing them into one table with nullable "target" and "actual" columns
looks tidier and makes every subsequent query worse — you can no longer ask "what did they
actually lift" without filtering out rows that were only ever intentions.

A `ScheduledSession` may reference a template or not. Trainers improvise, and a session
built on the day is a normal session, not a degenerate one.

## Logged sets

```sql
logged_set (
  id                    BIGSERIAL PRIMARY KEY,
  scheduled_session_id  BIGINT NOT NULL REFERENCES scheduled_session,
  exercise_id           BIGINT NOT NULL REFERENCES exercise,
  set_number            SMALLINT NOT NULL,
  weight_kg             NUMERIC(6,2),
  reps                  SMALLINT,
  rpe                   NUMERIC(3,1),
  UNIQUE (scheduled_session_id, exercise_id, set_number)
)
```

`weight_kg` is nullable because bodyweight exercises have no weight, and `reps` is
nullable because timed holds have no reps. `rpe` is nullable because most trainers do not
use it and forcing a value would make the data worse, not better.

`NUMERIC`, not `DOUBLE PRECISION`. Weights land on 2.5 kg boundaries and float
representation error turns 62.5 into 62.499999999999996 in exactly the place a trainer
will notice it.

## Last-session lookup

The query the whole app is built around: *what did this athlete lift for this exercise,
last time they did it?*

The naive version fetches the athlete's sessions, loops, and queries per exercise — an
N+1 that gets slower every week the athlete keeps training. The correct version is one
query:

```sql
SELECT DISTINCT ON (ls.exercise_id)
       ls.exercise_id, ls.set_number, ls.weight_kg, ls.reps, ls.rpe, ss.session_date
FROM logged_set ls
JOIN scheduled_session ss ON ss.id = ls.scheduled_session_id
WHERE ss.athlete_id = :athleteId
  AND ss.session_date < :today
  AND ls.exercise_id = ANY(:exerciseIds)
ORDER BY ls.exercise_id, ss.session_date DESC, ls.set_number;
```

`DISTINCT ON` is PostgreSQL-specific and it is one of the reasons the database choice is
PostgreSQL rather than MySQL. Index: `(athlete_id, session_date DESC)` on
`scheduled_session`, and `(scheduled_session_id, exercise_id)` on `logged_set`.

## Measurements

```sql
measurement (
  id          BIGSERIAL PRIMARY KEY,
  athlete_id  BIGINT NOT NULL REFERENCES athlete,
  taken_on    DATE NOT NULL,
  notes       TEXT,
  UNIQUE (athlete_id, taken_on)
)

measurement_site (
  id             SMALLSERIAL PRIMARY KEY,
  code           VARCHAR(32) NOT NULL UNIQUE,   -- 'waist', 'left_arm'
  label          VARCHAR(64) NOT NULL,
  unit_kind      VARCHAR(8) NOT NULL,           -- MASS | LENGTH | PERCENT
  display_order  SMALLINT NOT NULL
)

measurement_value (
  id              BIGSERIAL PRIMARY KEY,
  measurement_id  BIGINT NOT NULL REFERENCES measurement ON DELETE CASCADE,
  site_id         SMALLINT NOT NULL REFERENCES measurement_site,
  value           NUMERIC(7,2) NOT NULL,
  UNIQUE (measurement_id, site_id)
)
```

**Why not one wide row.** A `measurement` table with `chest`, `waist`, `hips`,
`left_arm`, `right_arm`… works until the trainer wants to track one more site, and then
every new site is a schema migration. With a site table, adding "neck" is an `INSERT`.

**Why not a free-text key.** Because a `VARCHAR` the UI writes into produces `waist`,
`Waist` and `wasit` within a month, and every chart silently splits into three. The
reference table constrains the vocabulary without constraining the schema.

**Why a row per value rather than a JSON column.** Charting a site over time is
`WHERE site_id = ?` with an index, not a JSON path expression across every row the
athlete has ever had.

The cost of this shape is that reading a measurement session is a join and rendering a
history table is a pivot. That is a known, bounded cost, paid in one query each.

## Units

**Stored canonical, always.** Mass in kilograms, length in centimetres, percentages as
0–100. The trainer's metric/imperial preference is a display setting on the trainer
profile, applied at the edges — converted on the way out for rendering and on the way in
before it reaches the API.

There is no unit column next to a value anywhere in this schema, and that is deliberate.
A `180` that might be pounds or kilograms is unrecoverable after the fact; you cannot
infer it, and the person who could tell you has forgotten. See
[ADR 0004](decisions/0004-store-measurements-in-canonical-units.md).

## Migrations

Flyway, forward-only. An applied migration is never edited — a mistake is corrected by a
new migration, because the alternative is a checksum failure on every environment that
already ran it.

Seed data for `measurement_site` ships as a migration, not as application startup code,
so a fresh database and a migrated one are identical.
