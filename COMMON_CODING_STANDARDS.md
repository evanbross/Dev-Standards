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

## Git Workflow

- **Prefix temporary/working branches with `evan-`** (e.g. `evan-fix-duplicate-jobs`, `evan-2026-09-11-session`) so branch authorship is identifiable at a glance. Does not apply to branches the user explicitly names with a different prefix themselves.
- **Push to the remote immediately after every commit**, on any branch — not just `main`. Use `git push -u origin <branch>` the first time a new branch is pushed so it tracks upstream. If the push fails (e.g. no SSH agent in the terminal), report the failure clearly and tell the user to push manually (e.g. `! git push` in Claude Code, or via TortoiseGit) rather than silently leaving commits unpushed.

---

## Event-Driven Design Over Polling

Default to push/event-driven mechanisms (WebSockets, SSE, webhooks, pub/sub, filesystem watchers, DB triggers/LISTEN-NOTIFY) over polling loops whenever a component needs to react to state change owned by another component. Polling is cheap to write, but it imposes a constant tax even when nothing changed, and it caps responsiveness at the poll interval — the system is always at least "one interval" late by construction.

- Before adding a `setInterval`/`setTimeout` loop, a `while True: sleep(N)` loop, or a scheduled alarm/cron tick to detect a change, ask whether the producer of that change can instead notify the consumer directly.
- If a push mechanism is added, keep the underlying data store as the single source of truth — the push channel should carry a lightweight signal ("something changed, go check") rather than being the only path data can travel, so a dropped connection or a missed message never loses state, only adds latency.
- A slow, infrequent poll (minutes, not seconds) is still the right tool as a **fallback safety net** behind a push mechanism — it bounds the worst case if the socket/webhook is down — but it should not be the primary mechanism once a push path is available.
- Don't over-apply this: a value that changes only because of direct user action within the same request/response cycle, or a one-off status check the user explicitly triggers, doesn't need an event system built around it. This rule is about recurring background checks for state owned elsewhere.

**Why:** polling that's cheap at small scale becomes the dominant source of lag and wasted work as a system grows, and the interval becomes a hard floor on responsiveness that's easy to forget until someone asks why something feels slow.

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

## Do Not Cut Corners

When a fix or feature surfaces an existing shortcut — a misleading name, a duplicated
list, a validation step that was skipped, a root cause left unaddressed while a symptom
gets patched — fix the actual thing instead of routing around it, even when the narrow
patch is faster to ship. "It technically works" is not the bar; "it won't quietly mislead
the next person who reads or extends it" is.

This is not a license to over-build. It is the opposite failure mode from
**No speculative abstractions** above: that rule says don't build generality nobody asked
for; this rule says don't leave a known problem in place because fixing it properly costs
more than the minimum patch. Telling the two apart:

- **Speculative abstraction** (avoid): adding a config system, plugin architecture, or
  extra indirection for a hypothetical future need that doesn't exist yet.
- **Cutting a corner** (avoid): noticing a column/field is misnamed, a list is duplicated
  in two places, or a helper's logic is copy-pasted three times — and leaving it that way
  because the task at hand only strictly required touching one of the copies.

**How to apply:**
- If a rename, bug fix, or new feature touches code that has a sibling duplicate
  elsewhere (the same fallback expression, the same allowlist, the same magic string),
  fix all of them in the same change, not just the one the task happened to touch.
- If you find yourself hardcoding a name, path, or value that's already defined as a
  constant/enum/column list elsewhere, reference that source instead of retyping it —
  see **Single Source of Truth** below.
- If full scope is too large for the current change (e.g. dozens of call sites), it's fine
  to do the change in the highest-leverage places first — but say so explicitly to the
  user, naming what's deferred, rather than silently leaving a partial fix that looks
  complete.
- When genuinely blocked on scope or risk (a live production migration, a change that
  would touch code outside the current task's stated boundaries), surface the tradeoff
  and let the user decide — don't unilaterally decide "good enough" and move on quietly.

**Why:** a corner cut today reads as done. It isn't discovered as a shortcut until it
causes a real bug — two copies of a list drifting apart, a column name that no longer
describes what it holds, a duplicated fallback that one call site updates and the other
doesn't. The cost didn't go away; it moved downstream and got a compound interest rate.

---

## Single Source of Truth

Never duplicate authoritative data. If a set of valid values (status enums, stage keys, feature flags, config options) is already stored in a database table, config file, or other authoritative location, validation and business logic must read from that source at runtime — not from a parallel hardcoded copy.

**Why this matters:** Duplicate lists silently drift apart. When one copy is updated and the other is not, the bug is invisible until a value that exists in one place is rejected by the other.

**How to apply:**
- Before writing a validation constant (e.g. `VALID_FOO = frozenset({...})`), ask: is this set already authoritative somewhere?
- If yes, query or import from the authoritative source instead of copying the values.
- If a hardcoded copy already exists alongside an authoritative source, remove the copy and replace references with a runtime lookup.

---

## Verify Every Cross-Boundary Contract at Its Owning Layer

Any code that consumes a contract owned somewhere else must be checked against the
authoritative definition and tested at a layer capable of rejecting an invalid
assumption. This applies to both internal and external boundaries, including:

- database tables, views, functions, migrations, constraints, and result-column aliases;
- HTTP routes, methods, request/response schemas, status codes, and authentication;
- imported modules, exported symbols, constructor signatures, and callback interfaces;
- frontend/backend payloads and serialized field names;
- files, environment variables, command-line tools, queues, webhooks, and third-party APIs.

Before writing or changing a consumer:

1. Locate the authoritative producer/definition and verify every consumed name, type,
   shape, method, and semantic assumption exactly. Do not infer a view column from its
   source column, an API response from a nearby model, or an export from memory.
2. Add or run a contract-level test that crosses the real boundary. The test must be
   capable of failing for the mistake being guarded against. Linters, compilation,
   mocked repositories, and tests that replace the producer cannot validate the
   producer/consumer contract they bypass.
3. Keep unit tests for local logic, but do not present them as evidence that an
   unexecuted SQL query, HTTP exchange, import, serialization path, or external call is
   valid.

When a contract mismatch is discovered, treat it as evidence of a potentially repeated
failure pattern: search the repository for the same identifier, assumption, and sibling
consumers; correct affected existing paths in scope; and add regression coverage at the
owning boundary. If the audit or remediation is too broad or risky for the current task,
report the exact deferred scope instead of silently fixing only the observed occurrence.

**Why:** code can compile, lint, and pass fully mocked tests while still failing on its
first real interaction. Verification must occur where the contract is actually enforced.

---

## Auto-Selected Runtime Values Must Not Silently Diverge

When startup code auto-selects a runtime value instead of using a fixed configured one
— a free port scan, an available temp directory, an auto-assigned host/interface —
check whether any *other* subsystem has that same value hardcoded or independently
configured. If one does, silent auto-selection isn't safe.

The common case is a port fallback: the configured port is busy, so the server moves to
the next free one and "starts successfully" — while a reverse proxy, an OAuth redirect
URI, a dev-server proxy target, or docs still point at the original port. The server
looks healthy; every dependent is now silently talking to the wrong address.

**Why:** the failure looks like success. Nothing errors until a request depending on the
diverged value fails, and the link between that failure and "the port auto-incremented
at startup" is non-obvious to trace.

**How to apply:**
- Before adding auto-selection for a port, host, or path, grep every place that value is
  referenced (config files, `.env`, external redirect URIs, proxy config, setup docs).
- If a dependent is fixed and can't itself be auto-derived (e.g. an OAuth redirect URI
  registered with an external provider), fail fast instead of falling back.
- If every dependent *can* derive from the same source, do that instead of duplicating
  the selection logic — read the one bound value at startup, not a second constant.

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

The default templates above are marked "Personal Use Only / non-commercial." A project that is
itself a commercial ZombieTech LLC product (sold or operated as a paid product/service, not merely
used personally) uses the proprietary variants instead — same copyright holder, license terms
swapped to proprietary/all-rights-reserved: `Headers/FileHeader_Python_Proprietary.txt` and
`Headers/FileHeader_JS_Proprietary.txt`. A project's `AGENTS.md`/`CLAUDE.md` states explicitly which
pair applies; do not assume the default.

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
