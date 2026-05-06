# Codex Coding Standards

## Standards Files — Read These First

At the start of every session, re-read this file and every standards file listed in the
project's `AGENTS.md` profile before making any changes.

**MySitter project profile:**
- `COMMON_CODING_STANDARDS.md`
- `PYTHON_STANDARDS.md`
- `FASTAPI_STANDARDS.md`
- `DATABASE_STANDARDS.md`
- `REACT_STANDARDS.md`
- `TESTING_STANDARDS.md`
- `INFRASTRUCTURE_STANDARDS.md`

Other projects will have their own profile in their `AGENTS.md`.

---

## General Behavior Rules

Rules are numbered for stable reference in review notes. Do not renumber when adding.

| # | Rule |
|---|---|
| G0 | **Ask once before editing files** — after the user gives a task that may require code changes, ask a single confirmation such as "Is it ok for me to start editing files?" Do not begin edits before that confirmation. Once approved, continue without asking again unless there is unusual risk or a real blocker. |
| G1 | **Follow the file layout in the plan exactly** — no extra directories, no parallel implementations, no bonus files. |
| G2 | **Re-read standards at session start** — see `COMMON_CODING_STANDARDS.md`. |

---

## Python / FastAPI (Backend)

| # | Rule |
|---|---|
| B1 | **Async throughout the API** — `async def` routes, async SQLAlchemy sessions, async Redis calls. No sync blocking calls inside async functions. |
| B2 | **Never access model attributes not defined in the model** — check `Mapped` columns before referencing them anywhere. Referencing a field that doesn't exist raises `AttributeError` at runtime with no compile-time warning. |
| B3 | **Use pytest for Python tests** — write new Python tests as pytest-style test functions and fixtures. Do not introduce `unittest.TestCase` or `unittest.main()` unless modifying an existing unittest-only file and converting it is out of scope. |

---

## TypeScript / React (Web — `apps/web`)

| # | Rule |
|---|---|
| F1 | **Always use the `@/` alias in `apps/web`** — configured in `tsconfig.json` and `vite.config.ts`. Never use relative `../../` paths in web. |
| F2 | **`enabled` guard on conditional queries** — any `useQuery` that is only valid for a specific role or state must pass `enabled: <boolean>`. Never fire unconditionally. |
| F3 | **`QueryClientProvider` placement** — never call hooks in the same component that renders `QueryClientProvider`. Split into an outer provider component and an inner consumer. |
| F4 | **TanStack Query for all server state** — no manual `useEffect` data fetching. |
| F5 | **Components own UI only** — data fetching goes in custom hooks under `hooks/`. Components call hooks; they do not call `apiClient` directly. |
| F6 | **`ApiClient` namespaces are object-literal class properties** — not methods. |
| F7 | **Object params, not positional** — always match the TypeScript function signature. |
| F8 | **Viewport height on routes that hide the NavBar** — auth pages (`/login`, `/register`) hide the NavBar. Use `min-h-screen`, not `min-h-[calc(100vh-4rem)]`. |

---

## Mobile (Expo / React Native — `apps/mobile`)

| # | Rule |
|---|---|
| M1 | **Relative imports only** — do NOT use the `@/` alias in `apps/mobile`. Use `../` relative paths. |
| M2 | **Verify export names against source** — do not assume mobile export names match the web app. |
| M3 | **SecureStore calls are synchronous-style** — do not `await` SecureStore in contexts that don't support async. |

---

## Infrastructure / CI

| # | Rule |
|---|---|
| I1 | **This project uses GitLab CI** — all pipeline config goes in `.gitlab-ci.yml`. Never create `.github/workflows/`. |

---

## Codex-Specific Behavior Patterns

These are failure modes observed empirically from Codex runs. Check for them during the Claude review pass after every Codex run.

**C1. Verify text file encoding after generation**
After any file is generated, check its encoding with the `file` command. If it reports UTF-16, re-encode to UTF-8 before using it.

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

Most dangerous for `requirements.txt`, `.env*`, and any config file parsed as plain text.

**C2. Codex may use the actual codebase as ground truth when it disagrees with the plan**
When the plan spec and the actual implementation disagree, Codex tends to use what the code says. This is generally correct — but always verify by reading the actual endpoint/method definition.

**C3. Codex writes workaround bootstraps instead of fixing broken imports**
When the codebase has broken `__init__.py` files, Codex will detect the import failures and work around them by injecting fake modules into `sys.modules`. Look for `conftest.py` containing `sys.modules["app.something"] = ...` or `types.ModuleType(...)`. Fix by correcting the actual `__init__.py` files.

**C4. Cross-reference all env var references against `.env.example` after generation**
When Codex generates a new `.env.example`, grep all `import.meta.env.VITE_*` and `process.env.*` references and confirm every one appears in the example file.

```bash
grep -rh "import\.meta\.env\.VITE_" apps/web/src/ | grep -o 'VITE_[A-Z_]*' | sort -u
grep -rh "process\.env\.EXPO_PUBLIC_" apps/mobile/ | grep -o 'EXPO_PUBLIC_[A-Z_]*' | sort -u
```

**C5. Retry standards file reads with resolved absolute paths**
If a standards file listed in `AGENTS.md` fails to read via a relative path, do not
immediately report it missing. First run `Resolve-Path` / `realpath` from the project
root and retry reading the resolved absolute path. Only report the file missing if both
the relative read and resolved-path read fail.

This guards against sandbox path translation issues where paths such as
`../Dev-Standards/...` may be incorrectly resolved outside the actual project tree.

---

## Known Codex Risk Patterns (Review Checklist)

| Risk | What to check |
|---|---|
| **Sandbox overconfidence** | Code marked "tested and working" only ran in Codex's isolated sandbox. Verify imports, paths, and env vars against the actual project. |
| **Dependency drift** | Packages Codex installed in its sandbox may be at different versions than `requirements.txt` / `package.json`. Always check manifests. |
| **Path assumptions** | Hardcoded absolute paths or relative paths that only work from Codex's working directory. |
| **Intermediate artifacts** | Leftover scratch files, commented-out attempts, or TODO stubs. |
| **Test environment bleed** | Tests that pass because Codex seeded its own DB. Verify they work against a fresh schema. |
| **File encoding** | Codex has previously saved files as UTF-16. Verify text files are UTF-8, especially `requirements.txt`. |

---

## Comparison vs. Gemini

| Dimension | Codex | Gemini |
|---|---|---|
| Missing files | Rare | Common |
| Logic bugs | Low | High (3–9 per phase) |
| Over-engineering | Low | High |
| Encoding/artifact issues | UTF-16 file encoding | None observed |
| HTTP method accuracy | Reads actual code | Follows plan spec |
| Test structure | Well-structured fixtures | Hardcoded IDs / improper mocking |
