# Database Standards

Rules for data access layers, migrations, and spatial data. Read `COMMON_CODING_STANDARDS.md` first.

---

## Connection Lifecycle

Every database connection must be closed in a `finally` block or via a context manager.
Never pair `.open()` / `.close()` manually without a `finally` — an exception between
them leaks the connection and, under Postgres, eventually exhausts the connection pool.

**Flask route handlers** — use a context manager that yields repos and closes in `finally`:

```python
# backend/utils.py
from contextlib import contextmanager

@contextmanager
def get_repos():
    repos = open_repositories(config.DATABASE_URL)
    try:
        yield repos
    finally:
        repos["conn"].close()

# route handler
@bp.route("/jobs/<int:job_id>")
def get_job(job_id: int):
    with get_repos() as repos:
        job = repos["jobs"].get_by_id(job_id)
        if not job:
            return jsonify({"error": "Not found"}), 404
        return jsonify(job)
```

If a route needs to release the connection before slow I/O (e.g. an AI call), end the
`with get_repos()` block before that work begins instead of calling `.close()` manually —
do the DB reads inside the block, then run the slow call after it exits.

**Background threads** — use an explicit `try/finally`, and release the connection around
slow external I/O the same way a route handler would (see `PYTHON_STANDARDS.md` §
Connection Lifetime vs. Slow External Work). A background thread still holds a pooled
connection, so the same exhaustion risk applies — "it's a thread, not a request" is not an
exemption:

```python
def _run_scan() -> None:
    repos = open_repositories(config.DATABASE_URL)
    try:
        candidates = repos["jobs"].claim_pending(limit=20)  # atomic claim — see below
        repos["conn"].commit()
    except Exception as exc:
        repos["conn"].rollback()
        logger.exception("Scan failed")
        return
    finally:
        repos["conn"].close()

    for job in candidates:
        result = scrape(job["url"])          # slow external I/O — no connection held here
        _record_scan_outcome(job["id"], result)  # reopens a connection only to write the outcome
```

Do NOT rely on the happy path to close the connection, and do NOT import or call
a private `_get_repos` function from route handlers — it bypasses the lifecycle guarantee.

**Claiming a row and delivering on it are separate writes.** A claim (marking a row so no
other worker/tick picks it up) must commit before any slow or uncertain external call
(email, third-party API) — see `PYTHON_STANDARDS.md` § SELECT-then-Write Race Conditions
for making the claim itself atomic. But a committed claim does not mean the external call
happened: if it fails, or the process dies between the two, the row is claimed and the
outcome is unknown. Record the delivery outcome in its own write, after the external call
returns, so a crash between claim and delivery leaves a claimed-but-undelivered row that a
reconciliation pass (or the next tick) can detect — instead of a claim silently standing in
for "delivered." See `TESTING_STANDARDS.md` § Failure Ordering and Uncertain Delivery for
the test this requires.

---

## Database Abstraction

Always introduce an abstraction layer between business logic and the database driver/ORM.
The goal is to make swapping providers (SQLite → local Postgres → managed cloud Postgres,
or back) a change confined to the infrastructure layer, not scattered through application code.

- **Repository pattern**: data access lives in repository classes (e.g. `UserRepository`,
  `BookingRepository`) with well-defined method signatures. Routers and services call
  repository methods — never raw queries directly.
- **Dependency-injected sessions/connections**: pass the DB session or connection into
  repositories via constructor or function parameter rather than importing a global. This
  makes it easy to swap the underlying connection at the call site (e.g. for tests or
  provider migration).
- **No provider-specific leakage into business logic**: things like `psycopg2` connection
  strings, SQLite pragmas, or cloud-provider SDK calls belong in the infrastructure/config
  layer, not in services or routers.
- **Abstract base or protocol where substitution is real**: define an abstract base class
  or `Protocol` when a repository actually needs more than one implementation — swapping
  providers, a test double standing in for infrastructure that can't run in CI, or an
  explicit multi-backend boundary. Don't add one by default for a repository with a single
  implementation and no concrete plan for another — that's the speculative abstraction
  `COMMON_CODING_STANDARDS.md` § No Speculative Abstractions says to avoid. The
  SQLite → Postgres provider swap this section is written for is itself a real,
  concrete substitution need, which is why it's listed here.

This applies to every persistence store, not just the primary DB — caches (Redis), object
storage, and search indexes included.

---

## Alembic / Migrations

- **Migration chain**: `down_revision` must point to the current head before running
  `upgrade`. Verify the chain with `alembic current` / `alembic heads` before applying.

- **JSONB array columns**: always set `server_default='[]'` on nullable JSONB array
  columns so existing rows are not `NULL` after the migration runs.

- **Backfill new per-entity tables**: when adding a table that must have one row per
  existing entity (e.g. one `CustomerProfile` per `CUSTOMER` User), include a backfill
  `INSERT ... ON CONFLICT DO NOTHING` in `upgrade()` immediately after `create_table`.
  Never leave existing entities without their required related row.

---

## GeoAlchemy2

When using GeoAlchemy2's `Geography` or `Geometry` column types, **do not add an explicit
`op.create_index` for the spatial column in Alembic migrations.** GeoAlchemy2 automatically
creates a GIST index on the column when the table is created, so Alembic's autogenerated
index will duplicate it and raise `DuplicateTable` at migration time.

If `alembic revision --autogenerate` emits a `create_index` for a spatial column, remove
it (and the corresponding `drop_index` in `downgrade`) before running `upgrade head`.
