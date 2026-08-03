# Gemini Coding Standards

## Standards Files — Read These First

At the start of every session, re-read this file and every standards file listed in the
project's agent config before making any changes.

---

## General Behavior Rules

Numbers are stable references — do not renumber when adding.

| # | Rule |
|---|---|
| G0 | **Re-read standards at session start** — see `COMMON_CODING_STANDARDS.md`. |
| G1 | **Follow the file layout in the plan exactly** — no extra directories, no parallel implementations, no bonus files. |
| G2 | **AI attribution** (in-code `# ai:` comments and `AI-Model:` git trailers): see `COMMON_CODING_STANDARDS.md` § AI Attribution. |

---

## Frontend — React / TypeScript

Gemini-specific failure modes observed on this stack. The base rules are in `REACT_STANDARDS.md`.

**F1. Verify all `@/` import aliases resolve to real files**
After generating any file that uses `@/lib/`, `@/hooks/`, or `@/components/` imports,
confirm the target file exists under the project's source directory (e.g. `apps/web/src/` or `src/`).
Gemini sometimes imports from `@/lib/api` when the actual file is `@/lib/apiClient`, or
references a component under a slightly wrong name. These fail at runtime — not at build time.

**F2. Presigned URL content-type must match actual file**
When calling an upload presign endpoint, pass the actual file's content type from
`file.type`. Never hardcode `"image/jpeg"` for a file input that accepts `image/*`.

**F3. Import hygiene after deletions**
When a method is removed from a service, simultaneously remove every import of that
method in every file that referenced it. Never leave stale imports.

**F4. TanStack Query rules**
`enabled` guard on conditional queries, `QueryClientProvider` placement: see
`REACT_STANDARDS.md` § TanStack Query.

---

## Backend — Python / FastAPI

Gemini-specific failure modes. Base rules are in `PYTHON_STANDARDS.md` and `FASTAPI_STANDARDS.md`.

**B1. Never access model attributes that aren't in the model definition**
See `FASTAPI_STANDARDS.md` § Never Access Undefined Model Attributes.

**B2. Never use `datetime.utcnow` in SQLAlchemy column defaults**
Always use `lambda: datetime.now(timezone.utc)`:

```python
# Wrong
created_at = Column(DateTime, default=datetime.utcnow)

# Right
created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
```

---

## Infrastructure / CI

**I1. Use the correct CI platform**
See `INFRASTRUCTURE_STANDARDS.md` § CI Platform.
