# Codex Coding Standards

**Read `COMMON_CODING_STANDARDS.md` first.** All rules there apply to every Codex task.
This file contains Codex-specific behavior rules, project-specific overrides, and
observed Codex failure modes.

Include the rules in this file verbatim at the top of every Codex prompt.
Rules are grouped by area. Each rule has a short code (e.g. `B3`) for easy reference in review notes.
Numbers are stable references — do not renumber when adding.

---

## General / Architecture

| # | Rule |
|---|---|
| G0 | **Ask once before editing files** — after the user gives a task or question that may require code or file changes, ask a single confirmation such as "Is it ok for me to start editing files?" Do not begin edits before that confirmation. Once the user approves, continue through the needed edits without asking again for each file, folder, or routine access step unless there is unusual risk or a real blocker. |
| G1 | → **COMMON_CODING_STANDARDS.md** (no speculative abstractions) |
| G2 | → **COMMON_CODING_STANDARDS.md** (no bare `pass` in stubs) |
| G3 | → **COMMON_CODING_STANDARDS.md** (no `__import__()`) |
| G4 | → **COMMON_CODING_STANDARDS.md** (no dead branches) |
| G5 | → **COMMON_CODING_STANDARDS.md** (all imports must be used) |
| G6 | **Follow the file layout in the plan exactly** — no extra directories, no parallel implementations, no bonus files. |
| G7 | → **COMMON_CODING_STANDARDS.md** (downstream resource creation) |
| G8 | → **COMMON_CODING_STANDARDS.md** (dependency tracking) |
| G9 | → **COMMON_CODING_STANDARDS.md** (no name shadowing) |

---

## Python / FastAPI (Backend)

| # | Rule |
|---|---|
| B1 | → **COMMON_CODING_STANDARDS.md** (SQLAlchemy 2.x — `Mapped` / `mapped_column`) |
| B2 | → **COMMON_CODING_STANDARDS.md** (SQLAlchemy 2.x count pattern) |
| B3 | → **COMMON_CODING_STANDARDS.md** (FastAPI routes fully typed) |
| B4 | → **COMMON_CODING_STANDARDS.md** (Router `tags`) |
| B5 | → **COMMON_CODING_STANDARDS.md** (never import from app entry point in routers) |
| B6 | **Never access model attributes not defined in the model** — check `Mapped` columns before referencing them anywhere. Referencing a field that doesn't exist as a mapped column raises `AttributeError` at runtime with no compile-time warning. |
| B7 | **Async throughout the API** — `async def` routes, async SQLAlchemy sessions, async Redis calls. No sync blocking calls inside async functions. |
| B8 | → **COMMON_CODING_STANDARDS.md** (business logic layering — routers call services, never raw queries) |
| B9 | → **COMMON_CODING_STANDARDS.md** (upsert race condition) |
| B10 | → **COMMON_CODING_STANDARDS.md** (consistent validation normalization) |
| B11 | → **COMMON_CODING_STANDARDS.md** (type hints when touching code) |

---

## Database / Migrations

| # | Rule |
|---|---|
| D1 | → **COMMON_CODING_STANDARDS.md** (Alembic `down_revision` must point to current head) |
| D2 | → **COMMON_CODING_STANDARDS.md** (JSONB columns need `server_default='[]'`) |
| D3 | → **COMMON_CODING_STANDARDS.md** (no spatial index in Alembic for GeoAlchemy2 columns) |
| D4 | → **COMMON_CODING_STANDARDS.md** (backfill new per-entity tables) |

---

## TypeScript / React (Web — `apps/web`)

| # | Rule |
|---|---|
| F1 | → **COMMON_CODING_STANDARDS.md** (strict TypeScript — no `any`, no `// @ts-ignore`) |
| F2 | **Shared types only** — never redeclare API response types or request/update body shapes locally. All shared types belong in `packages/types` and must be imported from `@mysitter/types`. |
| F3 | **TanStack Query for all server state** — no manual `useEffect` data fetching. |
| F4 | **Components own UI only** — data fetching goes in custom hooks under `hooks/`. Components call hooks; they do not call `apiClient` directly. |
| F5 | **Always use the `@/` alias in `apps/web`** — configured in `tsconfig.json` and `vite.config.ts`. Never use relative `../../` paths in web. |
| F6 | **`enabled` guard on conditional queries** — any `useQuery` that is only valid for a specific role or state must pass `enabled: <boolean>`. Never fire unconditionally. |
| F7 | **QueryClientProvider placement** — never call hooks in the same component that renders `QueryClientProvider`. Split into an outer provider component and an inner consumer. |
| F8 | → **COMMON_CODING_STANDARDS.md** (StrictMode double-invoke guard) |
| F9 | → **COMMON_CODING_STANDARDS.md** (no stub `onClick` / `onPress`) |
| F10 | → **COMMON_CODING_STANDARDS.md** (JSX opening/closing tag casing) |
| F11 | **ApiClient namespaces are object-literal class properties** — not methods. |
| F12 | **Object params, not positional** — always match the TypeScript function signature. Pass named parameter objects — never positional arguments that happen to fit by accident. |
| F13 | **Viewport height on routes that hide the NavBar** — auth pages (`/login`, `/register`) hide the NavBar. Use `min-h-screen`, not `min-h-[calc(100vh-4rem)]`. |
| F14 | → **COMMON_CODING_STANDARDS.md** (no portals or notification providers inside `ErrorBoundary`) |
| F15 | → **COMMON_CODING_STANDARDS.md** (all hooks must appear before any early return) |

---

## Mobile (Expo / React Native — `apps/mobile`)

| # | Rule |
|---|---|
| M1 | **Relative imports only** — do NOT use the `@/` alias in `apps/mobile`. Use `../` relative paths. |
| M2 | **Verify export names against source** — do not assume mobile component/hook export names match the web app. Always check the actual source file. |
| M3 | **SecureStore calls are synchronous-style** — do not `await` SecureStore in contexts that don't support async (e.g. synchronous initializers). |

---

## Testing

| # | Rule |
|---|---|
| T1 | → **COMMON_CODING_STANDARDS.md** (test fixtures must match endpoint `response_model`) |
| T2 | → **COMMON_CODING_STANDARDS.md** (never mock the database in integration tests) |
| T3 | → **COMMON_CODING_STANDARDS.md** (tests must work against a fresh schema) |

---

## Infrastructure / CI

| # | Rule |
|---|---|
| I1 | **This project uses GitLab CI** — all pipeline config goes in `.gitlab-ci.yml`. Never create `.github/workflows/`. |
| I2 | → **COMMON_CODING_STANDARDS.md** (include all system dependencies in Dockerfiles) |

---

## Codex-Specific Behavior Patterns

These are failure modes observed empirically from Codex runs on the MySitter project. Watch for them during the Claude review pass after every Codex run.

**C1. Verify text file encoding after generation**
After any file is generated by Codex, check its encoding with the `file` command. If it reports UTF-16, re-encode to UTF-8 before using it.

```bash
file apps/api/requirements.txt
# "Unicode text, UTF-16, little-endian text" → re-encode:
python3 -c "
with open('requirements.txt', 'r', encoding='utf-16') as f:
    content = f.read()
with open('requirements.txt', 'w', encoding='utf-8', newline='\n') as f:
    f.write(content)
"
```

This is most dangerous for `requirements.txt`, `.env*`, and any config file that pip or a build tool parses as plain text.

**C2. Codex may use the actual codebase as ground truth when it disagrees with the plan**
When the plan spec and the actual implementation disagree, Codex tends to use what the code says rather than what the plan says. This is generally correct behavior — but always verify by reading the actual endpoint/method definition.

**C3. Codex writes workaround bootstraps instead of fixing broken imports**
When the existing codebase has broken `__init__.py` files, Codex will detect the import failures in its sandbox and work around them by injecting fake modules into `sys.modules`. The workaround is brittle and masks a production bug.

**What to look for:** A `conftest.py` that contains `sys.modules["app.something"] = ...`, `types.ModuleType(...)`, or manual attribute assignments like `app_pkg.models = ...`.

**How to fix:** Check every `__init__.py` in the package tree. Fix re-exports, then replace the bootstrap with plain `os.environ` injection before app imports.

**C4. Cross-reference all env var references against .env.example after generation**
When Codex generates a new `.env.example`, grep all `import.meta.env.VITE_*` and `process.env.*` references in the app and confirm every one appears in the example file. Codex commonly generates only the vars it just added and omits pre-existing ones.

```bash
grep -rh "import\.meta\.env\.VITE_" apps/web/src/ | grep -o 'VITE_[A-Z_]*' | sort -u
grep -rh "process\.env\.EXPO_PUBLIC_" apps/mobile/ | grep -o 'EXPO_PUBLIC_[A-Z_]*' | sort -u
```

---

## Known Codex Risk Patterns (Review Checklist)

| Risk | What to check |
|---|---|
| **Sandbox overconfidence** | Code marked "tested and working" only ran in Codex's isolated sandbox. Verify imports, paths, and env vars against the actual project. |
| **Dependency drift** | Packages Codex installed in its sandbox may be at different versions than `requirements.txt` / `package.json`. Always check manifests after a Codex run. |
| **Path assumptions** | Hardcoded absolute paths or relative paths that only work from Codex's working directory. |
| **Intermediate artifacts** | Leftover scratch files, commented-out attempts, or TODO stubs from Codex iterating toward a solution. |
| **Test environment bleed** | Tests that pass because Codex seeded its own DB. Verify they work against a fresh schema with real migrations. |
| **File encoding** | Codex has previously saved files as UTF-16. Verify text files are UTF-8, especially `requirements.txt`. |

---

## Comparison vs. Gemini

| Dimension | Codex | Gemini |
|---|---|---|
| Missing files | Rare — follows file lists in plans closely | Common — skips infrastructure/CI steps |
| Logic bugs | Low | High (3–9 per phase) |
| Over-engineering | Low — stays close to spec | High — adds parallel implementations, unsolicited abstractions |
| Encoding/artifact issues | UTF-16 file encoding | None observed |
| HTTP method accuracy | Reads actual code (good) | Follows plan spec (may be wrong if plan is stale) |
| Test structure | Well-structured fixtures | Tends to use hardcoded IDs / improper mocking |

**General guidance:** Codex output requires fewer logic-level fixes but benefits from a mechanical file encoding check. Gemini output requires deeper architectural review.
