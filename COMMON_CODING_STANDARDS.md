# Common Coding Standards

Universal rules that apply to every AI coding assistant and every project.
Each project's agent config (e.g. `AGENTS.md`) lists which additional standards files apply.

---

## Session Startup

**Re-read your AI-specific standards file and every standards file listed in the project profile at the start of every session, before making any changes.**

Rules are added between sessions. A session that begins without re-reading the standards may operate on stale rules and introduce the exact problems the new rules were written to prevent. One read at session start is sufficient — you do not need to re-read before each individual edit.

---

## General Best Practices

When modifying code, follow established best practices for the language and framework in use:

- **SOLID principles and clean architecture** — single responsibility, open/closed, dependency inversion.
- **Proper separation of concerns** — UI, business logic, and data access in distinct layers.
- **Secure coding** — no injection risks, no overly broad exception catches, validate at system boundaries (user input, external APIs), trust internal guarantees.
- **Idiomatic use** — prefer stdlib/framework features over hand-rolled solutions.
- **Consistent style** — match naming and conventions already present in the codebase.
- **No speculative abstractions** — solve the problem at hand. Three similar lines is better than a premature abstraction.
- **Prefer editing existing files over creating new ones.**
- **OAuth / interactive auth flows must never be triggered from background API calls.** If a token is missing or invalid during a non-auth endpoint, raise an error and return it to the caller. Interactive auth flows belong exclusively in dedicated auth endpoints that the user explicitly invokes.

---

## Dependency Tracking

When a new third-party package (or package extra) is first imported in a project, verify it is listed in the project's dependency manifest and add it if missing:

- **Python**: `requirements.txt`, `pyproject.toml [project.dependencies]`, or both. Check extras too — `pydantic[email]` and `pydantic` are not the same entry.
- **Node**: `package.json` `dependencies` (runtime) or `devDependencies` (tooling/test-only). Do not add runtime packages under `devDependencies`.
- **Dev-only packages**: keep in a separate manifest (`requirements-dev.txt`, `devDependencies`) so they do not bloat production installs.
