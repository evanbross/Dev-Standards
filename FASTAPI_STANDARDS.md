# FastAPI / SQLAlchemy Standards

Rules for Python projects using FastAPI and/or SQLAlchemy. Read `COMMON_CODING_STANDARDS.md` and `PYTHON_STANDARDS.md` first.

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

## SQLAlchemy 2.x

Use `Mapped` / `mapped_column` for all ORM column definitions. Never use the legacy
`Column()` style — it is not compatible with SQLAlchemy 2.x's typing system.

For count queries, always use `select(func.count()).select_from(q.subquery())`.
Never use `.with_only_columns(func.count())` on a query that already has filters —
that's a 1.x pattern that silently drops WHERE clauses in 2.x.

---

## Async Throughout

`async def` route handlers, async SQLAlchemy sessions, async Redis/cache clients.
Do not mix sync blocking calls into async route handlers.

---

## Never Access Undefined Model Attributes

Before referencing `obj.some_field` on an ORM model, confirm the field exists as a
`Mapped` column or relationship. Do not invent convenience fields that would be useful
but aren't defined — they raise `AttributeError` at runtime with no compile-time warning.
