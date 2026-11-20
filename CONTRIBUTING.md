# Contributing to Pace

Thanks for wanting to help. This page gets you a working dev environment
and covers the few conventions that matter.

## Dev setup

Same as the README's [Quick start](readme.md#quick-start):

```bash
git clone https://github.com/manzar-shahaan/workoutguide.git
cd workoutguide
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
docker compose up -d                      # Postgres 16 on localhost:5432
python scripts/init_db.py                 # schema
python scripts/generate_access_code.py "dev"   # signup code
python run.py                             # http://localhost:5001
```

Two optional helpers once the app is running:

- `python scripts/seed_sample_data.py --user-id <id>` — realistic sample
  workouts so the dashboard and muscle map have something to show.
- `python scripts/smoke_db.py` — verifies the app can reach the database.

## Code layout

```
app/
  __init__.py        app factory + Flask-Login setup
  config.py          configuration defaults (DATABASE_URL, SECRET_KEY, cookies)
  db/
    schema.sql       the schema, single source of truth
    connection.py    SQLAlchemy engine + DATABASE_URL
    repositories/    SQL data access (users, workouts, exercise catalog, stats, ...)
  web/
    routes/          HTTP routes for auth, workouts, exercises, PWA
    templates/       Jinja templates
    static/          JS (body map, set lists, pickers, stats) + PWA assets
scripts/             DB init, migrations, imports, seeds, smoke tests
utils/               fixed vocabularies (tags, body regions)
tests/               tests
```

Repositories own SQL; routes stay thin; templates and static JS own the UI.

## Conventions

These are the rules that keep Pace Pace. Please read them before your
first PR.

- **Per-set data stays the source of truth.** Don't add writes of
  averaged-summary values anywhere the per-set rows should be read and
  aggregated instead. The rollup columns on `exercise` exist for
  quick-list display only.
- **Migrations must be re-runnable.** Every script in `scripts/` is
  idempotent — running it twice is a no-op the second time, not a
  duplicate or a crash. New migration scripts follow the same rule.
- **Tags and body regions are fixed, seeded vocabulary.** They're
  defined in `utils/tags.py` and `utils/body_regions.py` and are not
  user-editable at runtime. Don't add endpoints that mutate them; extend
  the seed lists if a vocabulary term is genuinely missing.
- **Write for a stranger's instance.** Scripts that mutate data must be
  safe against a database that isn't yours: scope them, default to a dry
  run, and never silently overwrite existing state. No personal data
  (your exercises, workouts, IDs) baked into source files.
- **Imperative commit messages.** One concise summary line: "Add
  per-interval distance rollups", not "Added some stuff".

## Reporting issues

Open a GitHub issue with what you did, what you expected, and what
happened. Include your Python and PostgreSQL versions and the relevant
script output (redact anything personal — it's a public repo).

## License

By contributing, you agree that your contributions are licensed under
the [AGPL-3.0](LICENSE), the same as the rest of the project.
