# Common Coding Standards

Rules shared across all AI coding assistants and all projects (Claude, Codex, Gemini).
Each assistant's per-project config file references this document.

---

## Code Style

### Python

- Always use type hints on function signatures and class attributes.
- Prefer OOP over procedural/functional style — use classes to encapsulate related state and behavior.
- Follow PEP 8 naming: `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_CASE` for constants.

### General

- Write clear, self-documenting code; add comments only where the logic isn't obvious.
- Don't add speculative abstractions — solve the problem at hand, not hypothetical future ones.
- Prefer editing existing files over creating new ones.

---

## Best Practices

When modifying code, always follow established best practices for the language and framework in use. This includes (but is not limited to):

- SOLID principles and clean architecture.
- Proper separation of concerns.
- Secure coding patterns (no injection risks, no overly broad exception catches, validate at boundaries).
- Idiomatic use of the language and framework — prefer stdlib/framework features over hand-rolled solutions.
- Consistent naming and style conventions already present in the codebase.
- **OAuth / interactive auth flows must never be triggered from background API calls.** If a token is missing or invalid during a non-auth endpoint, raise an error and return it to the caller — never open a browser window or start a local callback server mid-request. Interactive auth flows belong exclusively in dedicated auth endpoints (e.g. `POST /auth`, `GET /login`) that the user explicitly invokes.

---

## Python

### Imports

All imports go at the top of the file — never inside a function or route handler.
The only exception is a genuinely optional dependency guarded by `try/except ImportError`
where the code degrades gracefully without it. In that case the import belongs inside
the `try` block, not scattered through function bodies.

Never use `__import__()` — always add a normal import at the top of the file.
A dynamic import inside an expression is a sign something went wrong; stop and restructure.

All imports must be used. Remove any import that is not referenced in the file body.

### Name shadowing

Never reuse a name that is already bound in an enclosing scope — function parameters,
outer variables, imported names, loop variables, comprehension variables, `with` targets,
`except` clause variables, and nested functions alike. Pick a distinct name; silently
overwriting an outer binding is a latent bug regardless of whether the outer value is
still needed.

### Datetime

Always use `datetime.now(UTC)` — never `datetime.utcnow()`. The latter is deprecated
in Python 3.12 and emits `DeprecationWarning`. `UTC` is available from the `datetime` stdlib:

```python
from datetime import datetime, UTC
ts = datetime.now(UTC).isoformat(timespec="seconds")
```

### Exception handling

`except Exception` is only acceptable at the outermost boundary of a request handler or
background worker thread, where the alternative is crashing the entire process. In all
other locations, catch the specific exception types you expect. Network fetch code should
catch `(urllib.error.URLError, TimeoutError, socket.error, json.JSONDecodeError, ValueError)`
— not bare `Exception` — so that programmer errors (`TypeError`, `AttributeError`,
`KeyError`) fail loudly and appear in logs with a traceback.

### Stub implementations

Never use bare `pass` for stub implementations — always use
`raise NotImplementedError("not yet implemented")` instead. A stub that raises fails
loudly the moment it is hit; a `pass` silently does nothing and can go undetected for a
long time. The only acceptable use of `pass` is in genuinely empty blocks (e.g.
`except SomeError: pass` where swallowing is intentional).

### Dead code

No dead code branches — `if False:`, `if True:`, unreachable arms, commented-out
alternatives. If the right approach is unclear, pick one and commit to it.

### Business logic layering

Routers and request handlers validate input and return responses. Business logic lives in
service or domain classes. Raw queries never appear in routers — they go through
repositories or services. This is a concrete application of the separation of concerns
principle and is what makes individual layers independently testable.

### Circular imports — never import from the entry point

Never import shared infrastructure (rate limiters, DB sessions, config, app state) from
the application entry point (`main.py`, `app.py`, etc.) inside a router or sub-module.
The entry point imports all routers, so a router that imports back from it creates a
circular dependency that crashes startup. Shared infrastructure belongs in a `core/` or
`config/` module imported by both the entry point and the routers.

### Upsert race conditions

Any SELECT-then-INSERT upsert pattern is inherently racy under concurrent writes. The
correct pattern is to attempt the INSERT and handle the uniqueness error:

```python
try:
    session.add(new_obj)
    await session.flush()
except IntegrityError:
    await session.rollback()
    existing = await session.get(Model, pk)
```

Never assume the "row not found → INSERT" path is race-free.

### Consistent validation normalization

If one field in a Pydantic model (or any validation layer) normalizes its input
(e.g. uppercasing, stripping whitespace, lowercasing an enum value), apply the same
normalization to all similar fields in the same schema. Asymmetric validators are silent
footguns — a value stored in one canonical form will fail equality checks against the
same value stored in another form.

### SQL column-name allowlisting

Any repository method that builds a SQL statement by interpolating `dict.keys()` into the
identifier position (e.g. a generic `insert`/`update` helper) must validate those keys
against a class-level allowlist before building the statement. Values are always
parameterized; identifiers are always allowlisted.

```python
_ALLOWED_COLUMNS: frozenset[str] = frozenset({"title", "company", "url", ...})

def insert(self, row: dict) -> int:
    unknown = row.keys() - self._ALLOWED_COLUMNS
    if unknown:
        raise ValueError(f"Unknown columns: {unknown}")
    cols = ", ".join(row.keys())
    placeholders = ", ".join("?" * len(row))
    ...
```

### Repository transaction discipline

Repository methods must **not** call `commit()` themselves. Committing belongs in the
route or service layer so that multi-step writes are a single atomic transaction:

```python
# In a route handler:
repos["jobs"].insert(job_data)
repos["stages"].insert(stage_data)
repos["conn"].commit()   # one commit covers both writes
```

If the second write fails, the first is not yet committed and the connection can be rolled
back. This is especially important ahead of any database migration.

### File handles

Always use a `with` block for file handles, including third-party objects that support
the context manager protocol (e.g. `fitz.open(path)` for PyMuPDF). Never pair
`.open()` + `.close()` manually — an exception between them leaks the handle.

### SQLAlchemy 2.x

Use `Mapped` / `mapped_column` for all ORM column definitions. Never use the legacy
`Column()` style — it is not compatible with SQLAlchemy 2.x's typing system.

For count queries, always use `select(func.count()).select_from(q.subquery())`.
Never use `.with_only_columns(func.count())` on a query that already has filters —
that's a 1.x pattern that silently drops WHERE clauses in 2.x.

### Downstream resource creation

When writing a registration or creation flow, explicitly account for all downstream
resource creation. If the architecture requires related rows in other tables when a new
entity is created, create them in the same transaction or raise `NotImplementedError`
with a clear comment if intentionally deferred. Never silently omit them.

---

## Dependency Tracking

When a new third-party package (or package extra) is first imported in a project, verify
it is listed in the project's dependency manifest and add it if missing. This applies to:

- **Python**: `requirements.txt`, `pyproject.toml [project.dependencies]`, or both. Check
  for extras too — `pydantic[email]` and `pydantic` are not the same entry; the base
  package name alone does not pull in the extra.
- **Node**: `package.json` `dependencies` (runtime) or `devDependencies` (tooling/test-only).
  Don't add runtime packages under `devDependencies`.
- **Dev-only packages** (test runners, linters, type stubs): keep them in a separate
  manifest (`requirements-dev.txt`, `devDependencies`) so they don't bloat production installs.

---

## FastAPI / OpenAPI Completeness

Every FastAPI route endpoint must be fully typed so OpenAPI documentation is accurate and
request validation is enforced:

- **`response_model`**: every `@router.<method>` should declare `response_model=<PydanticModel>`
  (or rely on the return type annotation when FastAPI can infer it). Never leave it
  unspecified for non-trivial responses.
- **Request bodies**: always a typed Pydantic model, never `body: dict`. Raw `dict` skips
  validation and generates no schema in OpenAPI docs.
- **Router `tags`**: set `tags=[...]` on each `APIRouter` (not per-route) so all endpoints
  are grouped correctly in `/docs`.
- **`status_code`**: set the correct HTTP status on creation routes (e.g. `status_code=201`
  for POST that creates a resource).

When adding or modifying endpoints, check all four of the above before finishing.

---

## Database Abstraction

When creating data access layers, always introduce an abstraction layer between business
logic and the database driver/ORM. The goal is to make swapping providers (SQLite →
local Postgres → managed cloud Postgres, or back) a change confined to the infrastructure
layer, not scattered through application code.

Concrete requirements:

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
- **Abstract base or protocol**: define an abstract base class or `Protocol` for each
  repository so that a mock, in-memory, or alternative-provider implementation can be
  substituted without changing callers.

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

---

## Windows Services

### pywin32

When writing a `win32serviceutil.ServiceFramework` subclass, `SvcDoRun` must call
`self.ReportServiceStatus(win32service.SERVICE_RUNNING)` as its **first line** — before
any config loading, logging setup, or other initialization. The SCM will kill the process
if it doesn't receive this acknowledgement within ~30 seconds.

Always install pywin32 services using the venv Python directly
(`.venv\Scripts\python.exe service.py install`), never via `uv run python`. The `uv run`
wrapper causes the SCM to register the wrong executable path.

The SCM does not add the script's directory to `sys.path` when launching a service. Any
service module that imports local files must explicitly prepend its own directory at the
very top of the file, before any local imports:

```python
sys.path.insert(0, str(Path(__file__).resolve().parent))
```

### NSSM

When configuring NSSM (`nssm install` / `nssm set Application`), always use **absolute
paths** for both the executable and the script. The SCM has no working directory context,
so relative paths like `.venv\Scripts\python.exe` will fail with "The system cannot find
the file specified."

---

## TypeScript

- **No `any` types** — all values must be typed. `any` defeats the type checker for every
  consumer of that value downstream.
- **No `// @ts-ignore`** — fix the underlying type error instead of suppressing it.
- **Shared types centrally** — never redeclare API response types or request body shapes
  locally in a component or hook. All shared types belong in a shared types package and
  must be imported from there.

---

## React / JavaScript

### Hooks must appear before any early return

Never place a `useState`, `useEffect`, `useMemo`, `useCallback`, or any other hook after
a conditional `return`. React requires hooks to be called in the same order on every
render; an early return that skips a hook causes a "Rendered fewer hooks than expected"
crash on the next route change. Move all hook calls to the top of the component, before
any guard clauses.

### No stub event handlers

Every `onClick` / `onPress` handler must be fully implemented. If a feature is not ready,
disable the element (`disabled={true}`) or remove it from the JSX. Never ship a dead
interactive element — it confuses users and hides missing functionality.

### StrictMode `useEffect` guard

Any `useEffect` that calls a mutating API (POST/PATCH/DELETE) must be guarded with a
`useRef(false)` flag set before the call and checked at the top:

```js
const hasFired = useRef(false);
useEffect(() => {
  if (hasFired.current) return;
  hasFired.current = true;
  // ... mutating call
}, []);
```

React 18 StrictMode double-invokes effects in development, so an unguarded mutating
effect will fire twice and produce duplicate writes.

### JSX tag casing

Match opening and closing tag names exactly, including casing.
A mismatch (`<MyComponent>` / `</mycomponent>`) causes a runtime crash with a misleading
error. Triple-check casing on all closing tags after any rename.

### No portals or notification providers inside `ErrorBoundary`

`<Toaster />`, modal roots, and similar portal elements must be siblings of the
`ErrorBoundary`, not children inside it. When the boundary catches an error and renders
its fallback, any portals inside it are unmounted, causing toast notifications to
disappear exactly when error feedback is most needed.

### `setInterval` / `setTimeout` ref pattern

Any interval or timeout stored in a ref must be:
1. Cleared before being reassigned (in event handlers that could fire multiple times).
2. Cleaned up in the `useEffect` return.
3. Guarded against "component unmounted during async" races when the interval is
   scheduled inside a `.then()` / async callback.

```js
useEffect(() => {
  let cancelled = false;
  fetchStatus().then((s) => {
    if (cancelled) return;          // guard: component may have unmounted
    if (s.running) {
      pollRef.current = setInterval(pollStatus, 1500);
    }
  });
  return () => {
    cancelled = true;
    clearInterval(pollRef.current);
  };
}, []);

// In an event handler that restarts polling:
function handleRunNow() {
  clearInterval(pollRef.current);   // clear any existing interval first
  pollRef.current = setInterval(pollStatus, 1500);
}
```

### Write calls must check `res.ok`

After any `fetch` that writes data (POST/PUT/DELETE), check `res.ok` before refetching
or updating UI state. Silently refetching on a 4xx/5xx produces stale data with no
user-visible error.

```js
const res = await fetch("/api/...", { method: "PUT", ... });
if (!res.ok) {
  window.alert(`Save failed: ${res.status}`);
  return;
}
await refetch();
```

### Numeric inputs must be validated before sending

`parseInt` / `parseFloat` return `NaN` for empty or non-numeric input. `NaN` serialises
to `null` in JSON (`JSON.stringify({n: NaN})` → `'{"n":null}'`). Validate before sending:

```js
const days = parseInt(value, 10);
if (!Number.isFinite(days) || days < 1) { window.alert("Enter a valid number."); return; }
```

### `window.confirm` — use the qualified form

Always write `window.confirm(...)`, never bare `confirm(...)`. The qualified form is
unambiguous to linters and consistent with how all other global browser APIs are called.

---

## Testing

### No mocked databases in integration tests

Integration tests must run against a real database instance, not mocks. A test suite that
passes against mocks but fails against the real schema provides false confidence — the
mock diverges from the real schema over time and the divergence goes undetected until
production.

### Test fixtures must match the actual response schema

Before writing a fixture that reads a field from a response, verify that field exists in
the declared `response_model` or actual return value. Do not assume nested data is
present without checking the schema. Reading a field that doesn't exist produces `None`
silently in some frameworks and `KeyError` in others.

### Tests must work against a fresh schema

Do not seed the test database inside a sandbox and leave the test hardcoded against that
state. Use fixtures and `conftest.py` (or equivalent) so the test suite works correctly
against a freshly-migrated, empty schema.

---

## Infrastructure

### Include all system dependencies in Dockerfiles

Before writing a Dockerfile, identify every system-level dependency required by the
Python packages in use (e.g. `libpq-dev` for asyncpg, `gdal-bin` + `libgdal-dev` for
GeoAlchemy2, `libmagic` for python-magic). A missing system dependency causes an
`ImportError` at container startup — not at build time — making it hard to diagnose.
