<p align="center">
    <img src="docs/logo.svg" width="96" alt="Pace logo — a green dumbbell"/>
</p>

<h1 align="center">Pace</h1>

<p align="center">
    A self-hosted workout tracker where every set is the source of truth.<br>
    Muscle-map, tags, and progression charts — your data, your server. ⚡
</p>

<p align="center">
  <a href="LICENSE"><img alt="License: AGPL-3.0" src="https://img.shields.io/badge/license-AGPL--3.0-blue.svg"></a>
  <img alt="Python 3.10+" src="https://img.shields.io/badge/python-3.10%2B-blue.svg">
  <img alt="Flask" src="https://img.shields.io/badge/built%20with-Flask-000000.svg?logo=flask">
  <img alt="PostgreSQL" src="https://img.shields.io/badge/PostgreSQL-16-4169E1.svg?logo=postgresql">
</p>

---

## Why Pace

Most fitness apps average your work away, lock your history in their
cloud, and hide the data model. Pace does the opposite:

- **Every set is a row.** Resistance work logs real per-set rows (weight × reps); endurance and HIIT log one row per interval (duration + distance). Progression charts are computed from the actual sets you did — never an averaged summary.
- **Your data lives in your Postgres.** Pace runs on your own server, on your own database. No third-party cloud, no export gate, no account you can lose.
- **The muscle map tells you the truth.** It's the first thing you see when you open the app, and regions you haven't trained in a while pulse — a built-in answer to "what have I been neglecting?"
- **Exercises are classified three ways, not one rigid category:** how you log it (metric type), what it is (tags), and which muscles it hits (body regions).
- **Units stay honest.** lb/kg for weights and mi/km for distance, with kg values normalized underneath so mixed-unit history still charts correctly.

## Features

**Logging**

- Workouts per day, multiple exercises per workout.
- Resistance: per-set weight × reps rows.
- Endurance/HIIT: per-interval duration + optional distance rows, so interval sessions get one row per interval.
- lb/kg and mi/km support; weight values normalized to kg for consistent progression.

**The muscle map**

- Tappable front/back body map narrows a shortlist of your own exercises plus fresh suggestions for the region you tapped.
- Regions you haven't trained in a while pulse — the map doubles as a "what needs work" prompt. [How the map works →](docs/MUSCLE_MAP.md)
- A Strength/Cardio switch jumps straight to the cardio form, since a timed activity has no regions to tap.

**Three-axis exercise classification**

- **Metric type** — *how you log it*: "Weights & reps" or "Time & distance". Drives which fields the logging form shows.
- **Tags** — *what it is*: a curated multi-select (Cardio, Strength, HIIT, Plyometrics, …). Many per exercise — badminton is Cardio + Agility + Plyometrics — so tag-based rollups sum correctly across differently-named exercises. Optional; untagged exercises log fine.
- **Body regions** — *which muscles*: tap in priority order, unlimited regions. Resistance exercises only, so a run never counts toward muscle freshness.

**Your own exercise catalog**

- Exercises are per-user templates. Rename or merge them from the "Manage exercises" page, which shows each one's metric type, tags, and regions.
- As-you-type suggestions pull from your catalog with last-logged date and last sets; unknown names are created on save.
- Optional suggestions from [wger.de](https://wger.de) (CC-BY-SA) with preview images, shown only for regions you haven't covered with your own history.

**Stats**

- Dashboard with a calendar view and per-region / per-exercise progression charts, computed from per-set data.

**Import & install**

- Legacy JSON import (see below).
- Installable PWA: launches straight to the home/log screen with persistent login.

## Quick start

Self-host in under five minutes. You need **Python 3.10+** and **PostgreSQL**.

```bash
# 1. Clone and install
git clone https://github.com/manzar-shahaan/workoutguide.git
cd workoutguide
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 2. Get a Postgres database running (docker-compose.yml ships one:
#    Postgres 16, user/pass/db = workoutguide, port 5432, persistent volume)
docker compose up -d

# 3. Create the schema
python scripts/init_db.py

# 4. Create a signup code, then start the app
python scripts/generate_access_code.py "me"
python run.py
```

Open http://localhost:5001, sign up with the access code, and log your first workout.

- The default connection string is
  `postgresql+psycopg2://workoutguide:workoutguide@localhost:5432/workoutguide`
  (`app/db/connection.py`). Running your own Postgres? Set `DATABASE_URL` to your
  own URL instead of using the bundled compose file.
- `SECRET_KEY` defaults to a dev value; set it in production.
- Prefer bare Postgres over Docker? Any recent PostgreSQL works — the compose
  file is just the fastest path.

## Importing old workouts

Moving history from another tracker or a plain JSON export:

```bash
python scripts/import_legacy.py --path legacy.json --user-id 1 --dry-run --report import_report.json
```

`--path` points at your JSON file (array, object, or NDJSON — all handled), `--user-id` is your Pace user, `--dry-run` previews without writing, and `--report` writes a JSON report of what would happen. Run it without `--dry-run` when the preview looks right. All options (`--unit`, `--tz`, `--strict`, `--dedupe`, `--source`) are documented in [docs/SCRIPTS.md](docs/SCRIPTS.md).

## Database setup & migrations

- **Fresh install:** `python scripts/init_db.py` applies the full schema from `app/db/schema.sql`. That's it — no migration scripts needed.
- **Existing database** (predating `schema.sql`): run the migration scripts in order. Every one is idempotent.
- Full per-script reference, including the recommended migration order: [docs/SCRIPTS.md](docs/SCRIPTS.md).

## Data model

The short version:

| Table | What it holds |
|-------|---------------|
| `workout` | One day's session — date, optional notes. |
| `exercise_catalog` | Your exercise templates: name, metric type, tags, regions. |
| `exercise_set` | The source of truth: one row per set (weight × reps) or per interval (duration + distance). |
| `tag` | Fixed descriptive vocabulary — Cardio, Strength, HIIT, Plyometrics, … |
| `body_region` | Fixed anatomical regions backing the muscle map. |

The full relational picture, including the derived-rollup rule and the junction tables: [docs/DATA_MODEL.md](docs/DATA_MODEL.md).

## Contributing

Bug reports and pull requests are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for setup and conventions. Please keep discussion friendly; the [Code of Conduct](CODE_OF_CONDUCT.md) applies.

## License

[AGPL-3.0](LICENSE). Free to self-host, use, and modify. If you run a modified version of Pace as a network service, you must offer its source to the people using it — that's the AGPL network clause, and it keeps Pace and its forks open no matter who runs them.
