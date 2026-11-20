# Data model

The full relational picture. The [README](../readme.md) keeps a short
summary; this page is the complete table-by-table reference. Schema lives
in `app/db/schema.sql`.

| Table | Purpose |
|-------|---------|
| `app_user` | Account data, weight unit preference, week start. |
| `workout` | Date, `user_id`, optional notes. One workout per day. |
| `exercise_catalog` | User-specific exercise names/templates, with `metric_type` (resistance/endurance) driving which fields a logged entry populates. |
| `tag` | Fixed, non-user-editable descriptive vocabulary (seeded from `utils/tags.py`). |
| `exercise_catalog_tag` | Which tags apply to a catalog entry (many-to-many, unordered) — the "what is this" classification, independent of metric_type and regions. |
| `exercise` | Logged entries with optional `exercise_name` and `exercise_catalog_id`. `weight_used`/`num_of_sets`/`avg_reps`/`max_reps`/`total_duration_seconds`/`total_distance`/`distance_unit` are derived rollups for quick-list display, not the source of truth. |
| `exercise_set` | Per-set weight/reps (resistance) or per-interval duration/distance (endurance) rows — the actual source of truth for volume, progression, and per-set/interval display. A given exercise only ever populates one pair, per its catalog `metric_type`. |
| `body_region` | Fixed, non-user-editable anatomical regions for the muscle-map (seeded from `utils/body_regions.py`, keyed to the body-highlighter SVG). |
| `exercise_catalog_region` | Which regions a given catalog exercise hits, with a `rank` (1 = primary, 2 = secondary, ...) set by tap order in the picker. Resistance exercises only — endurance entries carry no region tags. |
| `suggested_exercise` / `suggested_exercise_region` | Unlogged exercises imported from wger.de (CC-BY-SA), shown below your own history in the region shortlist. |

## The one rule

`exercise_set` is the source of truth. The rollup columns on `exercise`
exist for quick-list display only — they are derived, and per-set rows
always win. Code that treats an average or a total as more authoritative
than the underlying rows is a bug.

## Classification axes

An exercise in your catalog is described along three independent axes:

- **metric_type** — how you log it: `resistance` (weight × reps per set)
  or `endurance` (duration + optional distance per interval). One per
  exercise; drives which fields the form shows and which pair of columns
  `exercise_set` populates.
- **tags** — what it is: a curated multi-select (Cardio, Strength,
  Mobility, Plyometrics, Agility, Balance, Core, Power, Endurance,
  Steady state, HIIT, Intervals, Sprints — `utils/tags.py`). Many per
  exercise. Optional — untagged exercises log fine, they just won't
  surface in tag-based rollups.
- **body regions** — which muscles: tap in priority order, unlimited
  regions. Resistance exercises only; switching an exercise to endurance
  clears any region tags, since a run shouldn't count toward muscle
  freshness.

## Exercise suggestions

Exercise entry uses a free-text field. As you type, suggestions are
pulled from your full exercise catalog (name, last-logged date, last
sets) and the total count of stored exercises is shown. If the name
doesn't exist yet, it's created on save. Below your own history, the
region shortlist also surfaces unlogged wger-sourced suggestions.

## Notes vs. exercise names

Exercise `notes` remain intact and are not used as the primary exercise
name. `exercise_name` is for clean cataloging and stats; notes capture
workout-specific details.
