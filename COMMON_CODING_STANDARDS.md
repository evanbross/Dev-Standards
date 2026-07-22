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

## Event-Driven Design Over Polling

Default to push/event-driven mechanisms (WebSockets, SSE, webhooks, pub/sub, filesystem watchers, DB triggers/LISTEN-NOTIFY) over polling loops whenever a component needs to react to state change owned by another component. Polling is cheap to write, but it imposes a constant tax even when nothing changed, and it caps responsiveness at the poll interval — the system is always at least "one interval" late by construction.

- Before adding a `setInterval`/`setTimeout` loop, a `while True: sleep(N)` loop, or a scheduled alarm/cron tick to detect a change, ask whether the producer of that change can instead notify the consumer directly.
- If a push mechanism is added, keep the underlying data store as the single source of truth — the push channel should carry a lightweight signal ("something changed, go check") rather than being the only path data can travel, so a dropped connection or a missed message never loses state, only adds latency.
- A slow, infrequent poll (minutes, not seconds) is still the right tool as a **fallback safety net** behind a push mechanism — it bounds the worst case if the socket/webhook is down — but it should not be the primary mechanism once a push path is available.
- Don't over-apply this: a value that changes only because of direct user action within the same request/response cycle, or a one-off status check the user explicitly triggers, doesn't need an event system built around it. This rule is about recurring background checks for state owned elsewhere.

**Why this matters:** polling that's "cheap and easy" at small scale quietly becomes the dominant source of lag and wasted work as a system grows — every consumer re-checks on its own clock regardless of whether anything changed, and the interval becomes a hard floor on responsiveness that's easy to forget is there until someone asks why something feels slow.

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

---

## Single Source of Truth

Never duplicate authoritative data. If a set of valid values (status enums, stage keys, feature flags, config options) is already stored in a database table, config file, or other authoritative location, validation and business logic must read from that source at runtime — not from a parallel hardcoded copy.

**Why this matters:** Duplicate lists silently drift apart. When one copy is updated and the other is not, the bug is invisible until a value that exists in one place is rejected by the other.

**How to apply:**
- Before writing a validation constant (e.g. `VALID_FOO = frozenset({...})`), ask: is this set already authoritative somewhere?
- If yes, query or import from the authoritative source instead of copying the values.
- If a hardcoded copy already exists alongside an authoritative source, remove the copy and replace references with a runtime lookup.

---

## File Headers

Every new source file must begin with the copyright/license header for its language. Copy the appropriate template verbatim as the first block of the file — do not paraphrase or abbreviate it.

| Language                | Template                        |
|-------------------------|---------------------------------|
| Python                  | `Headers/FileHeader_Python.txt` |
| JavaScript / TypeScript | `Headers/FileHeader_JS.txt`     |
| Java                    | `Headers/FileHeader_Java.txt`   |
| HTML                    | `Headers/FileHeader_HTML.txt`   |
| AI prompts              | `Headers/FileHeader_Prompt.txt` |

---

## AI Attribution

AI assistants must leave attribution breadcrumbs in two places: **in-code comments**
for quick visual review, and **git commit trailers** for per-line accuracy via
`git blame`. Both are required; neither alone is sufficient.

---

### In-code comments — mark at the granularity of the change

Add an `# ai:` comment at the exact location of what you wrote or replaced. The
position of the comment must unambiguously identify the code it belongs to.

| What you did | Where to put the `# ai:` comment |
|---|---|
| Wrote a new function or class | Line immediately before `def` / `class` |
| Wrote a new block within an existing function | Line immediately before that block |
| Replaced specific lines within a function | Line immediately before the replaced lines |
| Changed a single line within a larger block | Trailing inline comment on that line |

**Format:** `# ai: <model-id> | <YYYY-MM-DD>`  
Use your exact model identifier and the actual date the work was done.

```python
# ai: codex-o4-mini | 2026-05-01
def process_job(job: dict) -> None:
    original_block = job["title"]

    # ai: claude-sonnet-4-6 | 2026-06-08
    date_found = datetime.now(UTC).date().isoformat()  # was: date.today()
```

**Rules:**

- **Never remove or overwrite a marker for code you did not touch.** Multiple `# ai:`
  markers can and should coexist in one function when different AIs contributed
  different parts. This is the correct and intended state.
- **Replace when rewriting.** If you are replacing lines that carry an existing `# ai:`
  marker, replace that marker with yours — the code it described is gone.
- **No stacking on the same block.** Two `# ai:` comments in sequence describing the
  same code block is always wrong. One replacement comment is correct.
- **Date must be the actual work date.** Never copy a date from another `# ai:`
  comment or use a placeholder. Use today's date.
- **Omit for trivial changes.** Pure whitespace fixes, adding a single import, or
  reordering arguments inside a block that already has an `# ai:` comment does not
  require a new comment.

---

### Git commit trailers — per-line attribution

Every commit produced by an AI must include an `AI-Model:` trailer identifying the
exact model, alongside any `Co-Authored-By:` line:

```
Fix UTC datetime in ingest paths

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
AI-Model: claude-sonnet-4-6
```

The `AI-Model:` value must be the model's precise identifier string
(e.g. `claude-sonnet-4-6`, `codex-o4-mini`, `gemini-2.5-pro`), not a display name.
`git blame <file>` followed by `git show <hash>` then gives exact, per-line
attribution for any line in the project.
