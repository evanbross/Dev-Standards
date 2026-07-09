# Python Coding Standards

Rules for all Python projects. Read `COMMON_CODING_STANDARDS.md` first.

---

## Style

- Always use type hints on function signatures and class attributes.
- Prefer OOP over procedural/functional style — use classes to encapsulate related state and behavior.
- Follow PEP 8: `snake_case` functions/variables, `PascalCase` classes, `UPPER_CASE` constants.
- Write clear, self-documenting code; add comments only where the logic is non-obvious.

---

## Imports

All imports go at the top of the file — never inside a function or route handler.
The only exception is a genuinely optional dependency guarded by `try/except ImportError`
where the code degrades gracefully without it.

Never use `__import__()` — always add a normal import at the top of the file.
A dynamic import inside an expression is a sign something went wrong; stop and restructure.

All imports must be used. Remove any import that is not referenced in the file body.

Never import a name that begins with `_` from another module. The leading underscore
signals that the name is private to its defining module — importing it from outside
bypasses that boundary and creates an invisible dependency. If you find yourself needing
to write `from some_module import _helper`, the fix is one of:

- Remove the underscore to make it public (if it genuinely belongs to the module's API).
- Move it to a shared utility module imported by both callers.

---

## Name Shadowing

Never reuse a name that is already bound in an enclosing scope — function parameters,
outer variables, imported names, loop variables, comprehension variables, `with` targets,
`except` clause variables, and nested functions alike. Pick a distinct name; silently
overwriting an outer binding is a latent bug regardless of whether the outer value is
still needed.

---

## Datetime

All internal timestamps and dates must be in UTC. Never use local-time sources.

| Wrong | Right |
|-------|-------|
| `datetime.utcnow()` | `datetime.now(UTC)` |
| `date.today()` | `datetime.now(UTC).date()` |
| `CURRENT_DATE` (PostgreSQL) | `(NOW() AT TIME ZONE 'UTC')::date` |

`datetime.utcnow()` is deprecated in Python 3.12 and emits `DeprecationWarning`.
`date.today()` and PostgreSQL's `CURRENT_DATE` return values in the process/server's
local timezone; near midnight in a non-UTC environment they may produce the wrong date.

`UTC` is available from the stdlib — no third-party import needed:

```python
from datetime import datetime, UTC
ts = datetime.now(UTC).isoformat(timespec="seconds")
today = datetime.now(UTC).date()
```

---

## Data Validation with Pydantic

Use Pydantic models to validate any data crossing a system boundary — API request/response
bodies, config/environment loading, and external API responses — instead of hand-rolled
dict checks (`if "field" not in data: ...`) or manual type coercion.

- Define a `BaseModel` per external data shape, with type-hinted fields — this is the same
  discipline as the type-hints rule above, just enforced at runtime instead of only by the
  type checker.
- Prefer Pydantic's validators (`field_validator`, `model_validator`) over ad hoc post-parse
  checks scattered through route/service code.
- In FastAPI projects this is already required by `FASTAPI_STANDARDS.md` (request bodies as
  typed models, `response_model` on every route). In Flask or plain-Python projects, validate
  manually at the top of the route handler / entry point: parse the raw dict into a Pydantic
  model, catch `pydantic.ValidationError`, and turn it into a sanitized 400-style response
  before the raw data reaches business logic — never let a `ValidationError` escape to a
  generic exception handler that might leak field internals.
- Prefer `pydantic-settings` (`BaseSettings`) over manual `os.environ.get()` calls scattered
  across a config module — it gives typed, validated, single-source-of-truth environment
  config instead of re-parsing and re-validating env vars ad hoc at each call site.
- Don't use Pydantic to re-validate internal-only data that never crosses a boundary (e.g.
  values already produced by your own ORM query). That's redundant defensive coding for
  input that can't be malformed — reserve it for actual boundaries: incoming requests,
  config/env, and third-party API responses.

---

## Exception Handling

`except Exception` is only acceptable at the outermost boundary of a request handler or
background worker thread, where the alternative is crashing the entire process. In all
other locations, catch the specific exception types you expect. Network fetch code should
catch `(urllib.error.URLError, TimeoutError, socket.error, json.JSONDecodeError, ValueError)`
— not bare `Exception` — so that programmer errors (`TypeError`, `AttributeError`,
`KeyError`) fail loudly and appear in logs with a traceback.

---

## Outbound Network Requests — SSRF Protection

Any server-side code that fetches a URL that is caller-supplied, configured, imported, or
derived from external content (email bodies, scraped HTML, webhook payloads) must validate
the resolved destination before connecting, and again on every redirect hop. Checking the
URL string alone is not enough — DNS can resolve an innocuous-looking hostname to a
loopback, private, link-local (including the `169.254.169.254` cloud metadata address),
multicast, or reserved address, and `urlopen`'s default redirect handling does not
re-validate the `Location` header before following it.

- Resolve the hostname (`socket.getaddrinfo`) and check every returned address with
  `ipaddress.ip_address(...).is_private / is_loopback / is_link_local / is_multicast /
  is_reserved / is_unspecified` before connecting.
- Subclass `urllib.request.HTTPRedirectHandler` (or the equivalent in your HTTP client) to
  re-run the same check on `newurl` before following a redirect.
- Default-deny internal ranges. If one call site has a legitimate reason to reach a local
  service (e.g. testing a locally-running Ollama/LLM endpoint), give that call site an
  explicit, narrow opt-in (e.g. an `allow_loopback` parameter) rather than weakening the
  check for every fetch in the codebase.

---

## Service Layer Exception Handling — Log and Re-raise

Service and repository methods that call external infrastructure (APIs, databases, file
systems) should catch the expected infrastructure exception, log diagnostic context, and
re-raise. They must not swallow the exception or decide the HTTP response code — that
belongs to the route handler.

```python
# In a service/repository method:
try:
    result = external_client.call(...)
except SomeInfrastructureError as exc:
    logger.warning("Context about what was being attempted: %s", exc)
    raise  # re-raise unchanged; route handler decides the response

# In the route handler:
try:
    data = service.do_thing()
except NotAuthenticatedError:
    return jsonify({"error": "Not authenticated"}), 401
except SomeInfrastructureError:
    return jsonify({"error": "External service error"}), 502
except Exception as exc:
    logger.warning("Unexpected error: %s", exc)
    return jsonify({"error": "Unexpected error"}), 500
```

Never return raw `str(exc)` to the client from a generic handler — it may expose
connection strings, internal paths, or other implementation details. Log the full
exception server-side and return a sanitized message.

---

## Stub Implementations

Never use bare `pass` for stub implementations — always use
`raise NotImplementedError("not yet implemented")` instead. A stub that raises fails
loudly the moment it is hit; a `pass` silently does nothing and can go undetected for a
long time. The only acceptable use of `pass` is in genuinely empty blocks (e.g.
`except SomeError: pass` where swallowing is intentional).

---

## Dead Code

No dead code branches — `if False:`, `if True:`, unreachable arms, commented-out
alternatives. If the right approach is unclear, pick one and commit to it.

---

## Business Logic Layering

Routers and request handlers validate input and return responses. Business logic lives in
service or domain classes. Raw queries never appear in routers — they go through
repositories or services. This is a concrete application of the separation of concerns
principle and is what makes individual layers independently testable.

---

## Circular Imports — Never Import from the Entry Point

Never import shared infrastructure (rate limiters, DB sessions, config, app state) from
the application entry point (`main.py`, `app.py`, etc.) inside a router or sub-module.
The entry point imports all routers, so a router that imports back from it creates a
circular dependency that crashes startup. Shared infrastructure belongs in a `core/` or
`config/` module imported by both the entry point and the routers.

---

## New Shared Helpers Must Fit the Existing Import Graph

Before adding a new function to an existing shared module (e.g. `backend/utils.py`), check
what that module already imports. If two or more call sites that need the new function are
themselves imported *by* that shared module, putting the function there creates a cycle
(shared module → call site → shared module). Give the new function its own leaf module
(no imports from other project modules) instead, and have the shared module and the call
sites both import from it.

---

## Upsert Race Conditions

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

---

## Consistent Validation Normalization

If one field in a Pydantic model (or any validation layer) normalizes its input
(e.g. uppercasing, stripping whitespace, lowercasing an enum value), apply the same
normalization to all similar fields in the same schema. Asymmetric validators are silent
footguns — a value stored in one canonical form will fail equality checks against the
same value stored in another form.

---

## SQL Column-Name Allowlisting

Any repository method that builds a SQL statement by interpolating `dict.keys()` into the
identifier position must validate those keys against a class-level allowlist before
building the statement. Values are always parameterized; identifiers are always allowlisted.

```python
_ALLOWED_COLUMNS: frozenset[str] = frozenset({"title", "company", "url"})

def insert(self, row: dict) -> int:
    unknown = row.keys() - self._ALLOWED_COLUMNS
    if unknown:
        raise ValueError(f"Unknown columns: {unknown}")
    cols = ", ".join(row.keys())
    placeholders = ", ".join("?" * len(row))
    ...
```

---

## Repository Transaction Discipline

Repository methods must **not** call `commit()` themselves. Committing belongs in the
route or service layer so that multi-step writes are a single atomic transaction:

```python
# In a route handler:
repos["jobs"].insert(job_data)
repos["stages"].insert(stage_data)
repos["conn"].commit()   # one commit covers both writes
```

If the second write fails, the first is not yet committed and the connection can be rolled
back.

---

## File Handles

Always use a `with` block for file handles, including third-party objects that support
the context manager protocol (e.g. `fitz.open(path)` for PyMuPDF). Never pair
`.open()` + `.close()` manually — an exception between them leaks the handle.

---

## Downstream Resource Creation

When writing a registration or creation flow, explicitly account for all downstream
resource creation. If the architecture requires related rows in other tables when a new
entity is created, create them in the same transaction or raise `NotImplementedError`
with a clear comment if intentionally deferred. Never silently omit them.
