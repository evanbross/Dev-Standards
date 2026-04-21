# Global Claude Code Preferences

## Response Style

Aim for a mid-ground level of explanation. Don't be terse — explain *why* decisions are made, not just *what* to do. This helps with learning. Avoid excessive verbosity or padding, but don't skip reasoning when it's useful.

## Code Style

### Python
- Always use type hints on function signatures and class attributes
- Prefer OOP over procedural/functional style — use classes to encapsulate related state and behavior
- Follow PEP 8 naming: `snake_case` for functions/variables, `PascalCase` for classes, `UPPER_CASE` for constants

### General
- Write clear, self-documenting code; add comments only where the logic isn't obvious
- Don't add speculative abstractions — solve the problem at hand, not hypothetical future ones
- Prefer editing existing files over creating new ones

## Code Review

After writing or modifying code, proactively flag any violations of the best practices defined below. Don't wait to be asked — a brief "a few things worth noting:" section at the end of a response is appropriate.

## Best Practices

When modifying code, always follow established best practices for the language and framework in use. This includes (but is not limited to):

- SOLID principles and clean architecture
- Proper separation of concerns
- Secure coding patterns (no injection risks, no overly broad exception catches, validate at boundaries)
- Idiomatic use of the language and framework — prefer stdlib/framework features over hand-rolled solutions
- Consistent naming and style conventions already present in the codebase
- **OAuth / interactive auth flows must never be triggered from background API calls.** If a token is missing or invalid during a non-auth endpoint, raise an error and return it to the caller — never open a browser window or start a local callback server mid-request. Interactive auth flows belong exclusively in dedicated auth endpoints (e.g. `POST /auth`, `GET /login`) that the user explicitly invokes.

## Project CLAUDE.md Propagation

When starting work in any project that has a `CLAUDE.md` file, check whether it contains each of the following sections. If any are missing, add them (matching the content from this global file) before doing any other work. This ensures rules are captured in every project's version-controlled instructions.

Sections that must be present in every project CLAUDE.md:
- **Best Practices** — language/framework rules, SOLID, secure coding, idiomatic patterns
- **Database Abstraction** — repository pattern, dependency-injected sessions, no provider leakage, abstract base/Protocol per repository

## Code Generation Quality Rules

These exist because I have generated bad code in the past — enforce them strictly:

- **Never use `__import__()`** — always add imports at the top of the file. A dynamic import inside an expression is a sign I got confused; stop and restructure.
- **No dead code branches** — `if False:`, `if True:`, unreachable arms, commented-out alternatives. If I'm unsure which approach is right, I should pick one and commit, not leave both in.
- **SQLAlchemy 2.x count pattern** — always use `select(func.count()).select_from(q.subquery())`. Never use `.with_only_columns(func.count())` on a query that already has filters — that's a 1.x pattern that breaks in 2.x.
- **When generating multiple files in one session** — pause between files and flag uncertainty rather than patching in place. A `# TODO: verify this pattern` comment is better than a working-looking hack.
- **Never use bare `pass` for stub implementations** — always use `raise NotImplementedError("not yet implemented")` instead. A stub that raises fails loudly the moment it is hit; a `pass` silently does nothing and can go undetected for a long time. The only acceptable use of `pass` is in genuinely empty blocks (e.g. `except SomeError: pass` where swallowing is intentional).
- **When writing a registration or creation flow, explicitly account for all downstream resource creation** — if the architecture requires related rows in other tables when a new entity is created (e.g. a `ProviderProfile` row when a PROVIDER `User` registers), create them in the same transaction or raise `NotImplementedError` with a clear comment if intentionally deferred. Never silently omit them.

## Dependency Tracking

When a new third-party package (or package extra) is first imported in a project, verify it is listed in the project's dependency manifest and add it if missing. This applies to:

- **Python**: `requirements.txt`, `pyproject.toml [project.dependencies]`, or both. Check for extras too — `pydantic[email]` and `pydantic` are not the same entry; the base package name alone does not pull in the extra.
- **Node**: `package.json` `dependencies` (runtime) or `devDependencies` (tooling/test-only). Don't add runtime packages under `devDependencies`.
- **Dev-only packages** (test runners, linters, type stubs): keep them in a separate manifest (`requirements-dev.txt`, `devDependencies`) so they don't bloat production installs.

Check this any time a new `import`/`require` appears for a package that wasn't already in the manifest.

## FastAPI / OpenAPI Completeness

Every FastAPI route endpoint must be fully typed so OpenAPI documentation is accurate and request validation is enforced:

- **`response_model`**: every `@router.<method>` should declare `response_model=<PydanticModel>` (or rely on the return type annotation when FastAPI can infer it). Never leave it unspecified for non-trivial responses.
- **Request bodies**: always a typed Pydantic model, never `body: dict`. Raw `dict` skips validation and generates no schema in OpenAPI docs.
- **Router `tags`**: set `tags=[...]` on each `APIRouter` (not per-route) so all endpoints are grouped correctly in `/docs`.
- **`status_code`**: set the correct HTTP status on creation routes (e.g. `status_code=201` for POST that creates a resource).

When adding or modifying endpoints, check all four of the above before finishing.

## Database Abstraction

When creating data access layers, always introduce an abstraction layer between business logic and the database driver/ORM. The goal is to make swapping providers (SQLite → local Postgres → managed cloud Postgres, or back) a change confined to the infrastructure layer, not scattered through application code.

Concrete requirements:
- **Repository pattern**: data access lives in repository classes (e.g. `UserRepository`, `BookingRepository`) with well-defined method signatures. Routers and services call repository methods — never raw queries directly.
- **Dependency-injected sessions/connections**: pass the DB session or connection into repositories via constructor or function parameter rather than importing a global. This makes it easy to swap the underlying connection at the call site (e.g. for tests or provider migration).
- **No provider-specific leakage into business logic**: things like `psycopg2` connection strings, SQLite pragmas, or cloud-provider SDK calls belong in the infrastructure/config layer, not in services or routers.
- **Abstract base or protocol**: define an abstract base class or `Protocol` for each repository so that a mock, in-memory, or alternative-provider implementation can be substituted without changing callers.

This applies to every persistence store, not just the primary DB — caches (Redis), object storage, and search indexes included.

## GeoAlchemy2 / Alembic Gotcha

When using GeoAlchemy2's `Geography` or `Geometry` column types, **do not add an explicit `op.create_index` for the spatial column in Alembic migrations.** GeoAlchemy2 automatically creates a GIST index on the column when the table is created, so Alembic's autogenerated index will duplicate it and raise `DuplicateTable` at migration time.

If `alembic revision --autogenerate` emits a `create_index` for a spatial column, remove it (and the corresponding `drop_index` in `downgrade`) before running `upgrade head`.

## Context Window Management

Proactively suggest a `/clear` or `/compact` when the conversation is getting long — specifically:
- After completing a multi-step implementation block (e.g., finishing a numbered phase step)
- When 5+ large files have been read in the session
- Before starting a new major phase or feature

Mention it briefly at the end of a response: e.g., "This is a good point to `/clear` before continuing with Step 4."

## pywin32 Windows Services

When writing a `win32serviceutil.ServiceFramework` subclass, `SvcDoRun` must call `self.ReportServiceStatus(win32service.SERVICE_RUNNING)` as its **first line** — before any config loading, logging setup, or other initialization. The SCM will kill the process if it doesn't receive this acknowledgement within ~30 seconds, producing "The service did not respond to the start or control request in a timely fashion."

Also: always install pywin32 services using the venv Python directly (`.venv\Scripts\python.exe ddns_service.py install`), never via `uv run python`. The `uv run` wrapper causes the SCM to register the wrong executable path.

Also: the SCM does not add the script's directory to `sys.path` when launching a service. Any service module that imports local files must explicitly add its own directory at the very top of the file, before any local imports: `sys.path.insert(0, str(Path(__file__).resolve().parent))`. Without this, the import fails silently with no Event Log entry.

## NSSM Windows Services

When configuring NSSM (`nssm install` / `nssm set Application`), always use **absolute paths** for both the executable and the script. The SCM has no working directory context, so relative paths like `.venv\Scripts\python.exe` will fail with "The system cannot find the file specified."

## Environment

- Platform: Windows 10
- Primary shell: bash (use Unix syntax — forward slashes, `/dev/null`, etc.)
- Common stack: Python (Flask), React (Vite), PostgreSQL/SQLite
