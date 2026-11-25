# DEPLOY-SMA — libredb-studio tester environment

Baseline: upstream `1.3.0` (`58adba536f0283fecb7b761053cb2401dd7187eb`).

## Quick Start

```bash
./tester-env deploy    # Build from source, start, wait for health
./tester-env seed      # Sample + deterministic admin workspace (saved queries, history)
./tester-env verify    # Admin+user login, sample listing, 1000-row query, workspace rows, RBAC
./tester-env reset     # Clean slate (container + /app/data volume)
./tester-env stop      # Stop, keep the data volume
```

Default URL: `http://localhost:13390/`. From a browser container:
`http://host.docker.internal:13390/`.

| Role | Credentials |
| --- | --- |
| admin | `admin@libredb.org` / `test-admin` |
| user (query execution only) | `user@libredb.org` / `test-user` |

## Seed data (baseline)

`./tester-env seed` waits for the boot-time SQLite sample copy, asserts its pinned
size, then replaces three storage collections of the admin account from
`tester-env-seed-data.json` with `PUT /api/storage/<collection>`:
`active_connection_id`, `saved_queries` and `history`. Each PUT replaces the whole
collection, so `seed` is idempotent and also removes anything a browser session
saved (the smoke run's `Employee count smoke` query does not survive it). The
manifest with stable requirement ids is `tester-env-seed.json`.

Theme: a people-analytics workspace on the built-in HR employee database.

- Admin connection `Sample (Employees)` (`seed:sqlite-embedded-sample`, sqlite)
  is the active connection on login; the second admin connection is
  `Sample (LibreDB)` (`seed:libredb-embedded-sample`, libredb). The lower role
  sees only the LibreDB sample.
- The employee sample is the digest-pinned `seed-assets/sqlite/employee.db`
  (815104 bytes) copied to `/app/data/sample-employees.db`: `employee` 1000 rows,
  `department` 9, `dept_emp` 1103, `dept_manager` 16, `salary` 9488, `title` 1470,
  and 802 current employees (`dept_emp.to_date = '9999-01-01'`).
- 3 saved queries, `connectionType` sqlite, fixed `updatedAt` dates:
  `Headcount by department` (2026-11-24), `Average salary by title`
  (2026-11-20), `Top 10 paid employees` (2026-11-24). Tags: headcount/monthly,
  payroll, payroll/review.
- 4 query-history runs on `Sample (Employees)`, fixed `executedAt`
  2026-11-24T09:12:41Z … 2026-11-24T16:05:50Z: three success runs (9, 6 and 10
  rows) and one error run `SELECT * FROM employees;` → `no such table: employees`.

Pinned results (also asserted by `./tester-env verify` through `/api/db/query`):

| Saved query | Rows | First row |
| --- | --- | --- |
| Headcount by department | 9 | Development 203 (… Marketing 34, total 802) |
| Average salary by title | 6 | Senior Staff, 258 employees, avg 78955 |
| Top 10 paid employees | 10 | Arno Kumaresan, Sales, 136004 |

Browser-visible checks after `reset → deploy → seed`: the SQL IDE opens on
`Sample (Employees)`; the sidebar lists the admin's two connections; the Saved
panel shows the three queries with a `sqlite` badge and their dates; the History
panel (Active Conn) shows four runs, one with the red error icon, filtering by
`error` leaves one row; loading `Headcount by department` fills the editor and
RUN returns 9 rows with Development at 203. Logging in as the lower role lists
only `Sample (LibreDB)` and empty Saved/History panels.


## What the environment pins

- Image built from this checkout by the repo's own `Dockerfile` (bun deps →
  node trixie builder → standalone runtime), via
  `docker-compose.tester-env.yml`. No prebuilt image.
- `STORAGE_PROVIDER=sqlite` at `/app/data/libredb-storage.db`, so accounts,
  connections and saved queries are server-side and reset with the volume.
- Fixed `JWT_SECRET` (41 chars; shorter values are fatal at boot) and
  `AUTH_COOKIE_SECURE=false` because the browser origin is plain HTTP.
- `ADMIN_*`/`USER_*` seed the accounts table on first boot only. Under
  sqlite/postgres, later env changes do nothing until
  `ADMIN_PASSWORD_RESET=true` or a volume reset.
- `SEED_ASSETS_DIR=/app/seed-assets`: the digest-pinned SQLite
  `employee.db` sample (815104 bytes; `employee` = 1000 rows) is copied into
  `/app/data` at boot and listed admin-only as `seed:sqlite-embedded-sample`.

## Volumes and reset

One named volume (`libredb-studio-data`, `<RUN_ID>`-suffixed when
`--run-id` is used) mounted at `/app/data`. `./tester-env reset` runs
`docker compose down -v`, which drops the SQLite store and the copied sample;
the next deploy re-seeds accounts from env and re-copies the sample.

## Caveats

- Any `src/` change needs a rebuild. The lockfile-keyed deps layer is cached;
  `next build` re-runs in full (~1 min on 28 cores, cold total ~4 min).
- Writes to the sample database persist in the volume until a reset.
- The seeded workspace is per account: the admin's saved queries and history live
  under its own storage owner key, so the lower role has its own (empty) lists.
  The Saved panel filters by the active connection's *type* and the History panel
  by its *id*, which is why the seed also pins the active connection.
- A browser that signs in for the first time with a leftover `libredb_*` local
  copy of another install migrates it before pulling; re-run `./tester-env seed`
  and reload to restore the canonical state.
- Audit timestamps, monitoring metrics and the agent/MCP pages are time- or
  config-dependent; keep them out of assertions. The agent only exists when
  `LLM_API_KEY` is set.
- Timestamps are fixed UTC values, but the History and Saved panels render them
  with the browser's local timezone (`date-fns` `format`). Assert the query text,
  status icons and row counts; assert the date column only where the browser
  timezone is known.
- No AI/LLM env is set, so AI features stay off the oracle path.

## Fork and baseline pin

- Canonical fork: `https://github.com/sma-rl-dev/libredb-studio`
  (`gh repo fork https://github.com/libredb/libredb-studio --org sma-rl-dev --remote=false`;
  no personal-namespace fork exists).
- Branch `tester-env-baseline` = upstream tag `1.3.0`
  (`58adba536f0283fecb7b761053cb2401dd7187eb`) plus one tooling commit that only
  adds `tester-env`, `docker-compose.tester-env.yml`, `tester-env-seed-data.json`,
  `tester-env-seed.json` and this file. `gh api
  repos/sma-rl-dev/libredb-studio/compare/1.3.0...tester-env-baseline` reports
  ahead 1 / behind 0 and exactly those five file names, so no upstream source,
  `package.json`, `bun.lock` or Dockerfile is modified and the upstream production
  Dockerfile remains the build path.
- Submodule: `apps/libredb-studio` in the corpus, `.gitmodules` URL
  `https://github.com/sma-rl-dev/libredb-studio.git`, branch
  `tester-env-baseline`, pinned to the immutable tip commit (never a floating
  branch).
- `app_cli` is `tester-env` (submodule-relative). All deployment tooling lives in
  this fork; nothing was written to a corpus-level `deployment/` directory.
- Image naming follows the corpus policy: `IMAGE_TAG` is honoured verbatim
  (default `tester-env-libredb-studio:dev`) and never contains `RUN_ID`, which
  scopes only the container, compose project and `libredb-studio-data` volume.

## Fork-integration verification (fresh submodule checkout)

Run from `apps/libredb-studio` after the prospect stack was stopped with its own
CLI (`./tester-env reset` in the prospect checkout, dropping container, volume and
project network; no raw docker commands):

```bash
./tester-env deploy   # rebuilt from the fork checkout, healthy in 2 s (cached layers)
./tester-env seed     # sample 815104 bytes + 3 saved queries + 4 history runs
./tester-env verify   # passed
```

`verify` output: `GET /` -> 307 (login redirect), admin login role `admin`,
`seed:sqlite-embedded-sample` -> `Sample (Employees)` (sqlite), employee table =
1000 rows, workspace = active connection + 3 saved queries + 4 history runs
(3 success, 1 error), pinned saved-query results (Development 203 / 9 rows,
Senior Staff 78955 / 6 rows, Arno Kumaresan 136004 / 10 rows), the seeded failed
statement replays as `QUERY_ERROR` `no such table: employees`, and the user role
lists only the LibreDB sample with the admin-only sample hidden.
