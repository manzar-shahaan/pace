# Scripts reference

Every script in `scripts/`, what it does, and when to run it. The
[README](../readme.md) covers the short version; this page is the full
reference.

All scripts run from the repo root against the database in `DATABASE_URL`
(default: `postgresql+psycopg2://workoutguide:workoutguide@localhost:5432/workoutguide`,
see `app/db/connection.py`).

```
python scripts/<name>.py
```

## Table of contents

- [Setup](#setup)
- [Migration order for existing databases](#migration-order-for-existing-databases)
- [Migrations](#migrations)
- [Data tools](#data-tools)
- [Maintenance & repair](#maintenance--repair)
- [Dev helpers](#dev-helpers)

## Setup

### `scripts/init_db.py`

Applies `app/db/schema.sql` to an empty database. This is all a fresh
install needs — if you start here, none of the migration scripts below
apply to you. It is for empty databases, not for altering existing ones.

### `scripts/smoke_db.py`

Quick DB connectivity check. Confirms the app can reach the database.
Useful after setting `DATABASE_URL` to verify you're pointed where you
think you are.

## Migration order for existing databases

Databases created before `schema.sql` became the single source of truth
upgrade by running migrations in this order. Each is idempotent
(re-runnable); each prints what it did.

| # | Script | Purpose |
|---|--------|---------|
| 1 | `migrate_exercise_real.py` | Weight unit columns + kg backfill |
| 2 | `migrate_exercise_catalog.py` | Exercise catalog table + links |
| 3 | `migrate_body_regions.py` | Body regions, suggested exercises |
| 4 | `migrate_exercise_sets.py` | Per-set rows, backfilled from rollups |
| 5 | `migrate_region_rank.py` | Region role → rank |
| 6 | `migrate_modality.py` | Modality/cardio_target columns |
| 7 | `migrate_drop_muscle.py` | Drop the old muscle tables |
| 8 | `migrate_cardio_sets.py` | Per-interval duration/distance columns |
| 9 | `migrate_tags.py` | metric_type + tags (phase 1 of the split) |
| 10 | `migrate_drop_modality.py` | Drop modality columns (phase 3) |

`import_wger_exercises.py` and `backfill_exercise_regions.py` slot in
after step 3 — see [Data tools](#data-tools).

## Migrations

### `scripts/migrate_exercise_real.py`

Adds weight unit columns and backfills kg.

### `scripts/migrate_exercise_catalog.py`

Adds the exercise catalog table and links on `exercise`. Superseded by
`schema.sql` for fresh installs; kept as a historical record for existing
DBs that predate it.

### `scripts/migrate_body_regions.py`

Adds `body_region` / `exercise_catalog_region` / `suggested_exercise*`
tables and seeds the 17 regions. Run once on an existing DB.

### `scripts/migrate_exercise_sets.py`

Adds `exercise_set` and backfills per-set rows from each exercise's
rollup columns (`num_of_sets` rows at `avg_reps`, peak set bumped to
`max_reps`). Re-runnable, no-op for exercises that already have set rows.

### `scripts/migrate_region_rank.py`

Converts `exercise_catalog_region.role` (text, capped at
primary/secondary) to `rank` (integer, unlimited regions per exercise).
Re-runnable.

### `scripts/migrate_modality.py`

Adds `exercise_catalog.modality` / `cardio_target`, backfills modality
from the old muscle category, makes `muscle_id` nullable, and tightens
the catalog's unique constraint to `(user_id, name)`. Refuses to tighten
the constraint if it finds the same exercise name under two different
muscles — resolve those first. Superseded by `migrate_tags.py` +
`migrate_drop_modality.py`; kept as a historical record for existing DBs
that predate the tag split. Re-runnable.

### `scripts/migrate_drop_muscle.py`

Drops `exercise_catalog.muscle_id`, `exercise_muscle`, and `muscle`
entirely. Run any time after `migrate_modality.py` has been applied and
the app has been running on the region/modality model — nothing reads
these anymore. Re-runnable.

### `scripts/migrate_cardio_sets.py`

Adds `exercise_set.duration_seconds` / `distance` / `distance_unit`
(per interval) and `exercise.total_duration_seconds` / `total_distance`
/ `distance_unit` (rollups), and drops the stale `NOT NULL` on
`exercise.weight_unit` (endurance rows have no weight unit). Re-runnable.

### `scripts/migrate_tags.py`

Phase 1 of the modality → metric_type + tags split. Adds the
`tag` / `exercise_catalog_tag` tables (seeded from `utils/tags.py`) and
`exercise_catalog.metric_type`, then backfills `metric_type` + tags from
the old `modality` / `cardio_target` columns (cardio → endurance + a
`[cardio]` tag, `cardio_target` folded in as its own tag, e.g.
`[cardio, hiit]`). Safe to run before any UI change. Re-runnable — the
backfill step is skipped once `migrate_drop_modality.py` has run.

### `scripts/migrate_drop_modality.py`

Phase 3 — drops `exercise_catalog.modality` / `cardio_target` entirely.
Run only after `migrate_tags.py` has been applied and the app has been
running on metric_type + tags. Re-runnable.

## Data tools

### `scripts/import_wger_exercises.py`

Pulls exercises from the public wger.de API into `suggested_exercise`
and mirrors a small preview image for each locally (into
`app/web/static/suggested/`). Re-runnable/idempotent; `--max N` to test
with a small batch first. wger content is CC-BY-SA.

### `scripts/backfill_exercise_regions.py`

Fuzzy-matches existing `exercise_catalog` rows (logged before the
muscle-map existed) against `suggested_exercise` to tag them with
regions. Run after `migrate_body_regions.py` and
`import_wger_exercises.py`; re-runnable.

### `scripts/import_legacy.py`

Imports legacy workout JSON. Supports multiple JSON shapes (array,
object, or NDJSON) and normalizes into a canonical workout format.
Parses weight values with lb/kg suffixes and extracts exercise names
from notes. Legacy free-text muscle names can't be reliably mapped onto
the fixed region taxonomy ("arms" alone doesn't say biceps vs. triceps
vs. forearm), so imported exercises land untagged by region and get
body-map tags later via the add/edit form. One exception: a legacy
"cardio" muscle bucket is recognized and classified as
`metric_type=endurance` + a `[cardio]` tag automatically.

Example:

```
python scripts/import_legacy.py --path legacy.json --user-id 1 --dry-run --report import_report.json
```

Options:

| Flag | Effect |
|------|--------|
| `--path` | Path to the legacy JSON file (required) |
| `--user-id` | User ID to attach imported workouts to (required) |
| `--dry-run` | Validate and preview changes without committing |
| `--report PATH` | Write a JSON import report to the given path |
| `--unit lb\|kg` | Default unit when missing (defaults to the user's preference) |
| `--tz NAME` | Timezone for datetime normalization, e.g. `America/Los_Angeles` |
| `--strict` | Fail on invalid values instead of skipping |
| `--dedupe` | Skip duplicate exercises on re-import |
| `--source` | Force `legacy`, `generic`, or `auto` mapping (default `auto`) |

Start with `--dry-run` and a `--report`; the report shows exactly what
would be written before anything touches the database.

## Maintenance & repair

### `scripts/fix_exercise_regions.py`

Batch-restores body-region tags on `exercise_catalog` rows that lost
them. Only touches entries that have been logged at least once, and only
when they carry zero region tags (or are named explicitly via `--set`).
Tag values come from `--set "name=slug,slug"` (authoritative) or from
wger's own anatomical data via the same fuzzy name match
`get_or_create()` uses, only above the match threshold — anything that
doesn't match confidently is reported and skipped rather than tagged
with something approximate.

```
# report only, writes nothing (start here)
python scripts/fix_exercise_regions.py

# back up current tagging first -- there is no other copy of it
python scripts/fix_exercise_regions.py --export regions.json

# write the proposed tags
python scripts/fix_exercise_regions.py --apply

# supply or correct tags yourself (repeatable; slug order = priority)
python scripts/fix_exercise_regions.py --apply --set "squats=quadriceps,gluteal"

# one exercise at a time
python scripts/fix_exercise_regions.py --only 41 --apply
```

A confident name match does not guarantee a sensible tag set — wger's
entry for a close-matching name may target different muscles than yours
does. Read the dry run before applying, and correct anything wrong with
`--set` rather than accepting it.

### `scripts/report_tagging_status.py`

Read-only report of `exercise_catalog` classification — resistance
exercises missing body-region tags (actionable, since those won't show
up in the muscle-map shortlist), plus an informational count of
exercises with no descriptive tags (tags are optional, so this isn't a
problem to fix, just visibility).

## Dev helpers

### `scripts/seed_sample_data.py`

Optional seed data: tags and regions on resistance exercises, a
`[cardio]` tag plus duration/distance on endurance ones. Takes
`--user-id` or `--email` to target a user, and `--seed` for a
reproducible random sequence.

### `scripts/generate_access_code.py`

Generates a signup access code. Signup requires a valid, unused code, so
this is how a new account gets created:

```
python scripts/generate_access_code.py "your name"
```

The label is stored with the code so you can track who it was issued to.
Re-running the same label returns the existing code instead of a second
one.
