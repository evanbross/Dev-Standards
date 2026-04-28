# Common Coding Standards

Universal rules that apply to every AI coding assistant and every project.
Each project's agent config (e.g. `AGENTS.md`) lists which additional standards files apply.

---

## Session Startup

**Re-read your AI-specific standards file and every standards file listed in the project profile at the start of every session, before making any changes.**

Rules are added between sessions. A session that begins without re-reading the standards may operate on stale rules and introduce the exact problems the new rules were written to prevent. One read at session start is sufficient — you do not need to re-read before each individual edit.

---

## Context Health / Session Reset

If the assistant is likely to be operating with degraded context, it must tell the user
that `/compact` or `/clear` may be appropriate before continuing substantial work.

Examples include:
- repeated misses of simple references or project rules
- confusion about recently changed files
- losing track of active requirements
- tool outputs or conversation history becoming too large to reason about reliably
- a task expanding across many files after several rounds of edits

The assistant should state which option fits:
- `/compact` when the current task should continue but the conversation is getting large.
- `/clear` when starting a new unrelated task would be safer with a fresh session.

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

## Logging Format

Always configure the logging format to include **filename and line number** alongside the timestamp, level, and logger name. This is the single highest-value change for debugging — a stack trace tells you what crashed, but the log line tells you what was happening right before it.

**Python**
```python
logging.basicConfig(
    format="%(asctime)s [%(levelname)s] %(name)s (%(filename)s:%(lineno)d): %(message)s"
)
```

**Node / TypeScript** (Winston example)
```typescript
format: winston.format.printf(({ level, message, timestamp, filename, line }) =>
  `${timestamp} [${level}] (${filename}:${line}): ${message}`)
```

**Java** (Logback `logback.xml`)
```xml
<pattern>%d{HH:mm:ss} [%-5level] %logger{36} (%file:%line): %msg%n</pattern>
```

Use the bare filename (`%(filename)s` in Python, `%file` in Logback), not the full path — it keeps lines readable without losing the critical "where" context.

---

## Dependency Tracking

When a new third-party package (or package extra) is first imported in a project, verify it is listed in the project's dependency manifest and add it if missing:

- **Python**: `requirements.txt`, `pyproject.toml [project.dependencies]`, or both. Check extras too — `pydantic[email]` and `pydantic` are not the same entry.
- **Node**: `package.json` `dependencies` (runtime) or `devDependencies` (tooling/test-only). Do not add runtime packages under `devDependencies`.
- **Dev-only packages**: keep in a separate manifest (`requirements-dev.txt`, `devDependencies`) so they do not bloat production installs.
