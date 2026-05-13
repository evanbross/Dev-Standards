# Codex Review Checklist

Read this before the final response after code edits, when doing a review, or when
changing tests, imports, env config, dependencies, generated files, CI, or project
structure. This file preserves Codex-specific lessons from prior generated-output reviews.

## Review Triggers

Load this checklist when any of these are true:

- Code, tests, config, dependency manifests, env files, or generated files changed.
- The task involved debugging failed Codex output.
- The task involved imports, package layout, route wiring, model fields, or API contracts.
- The final answer will say something was tested or verified.

## General Behavior Rules

Rules are numbered for stable reference in review notes. Do not renumber when adding.

| # | Rule |
|---|---|
| G0 | Ask once before editing files when the project requires confirmation. Once approved, continue without asking again unless there is unusual risk or a real blocker. |
| G1 | Follow the file layout in the plan exactly: no extra directories, no parallel implementations, no bonus files. |
| G2 | Re-read standards at session start; see `COMMON_CODING_STANDARDS.md`. |

## Python / FastAPI Backend Checks

| # | Rule |
|---|---|
| B1 | Async FastAPI code must stay async throughout: async routes, async SQLAlchemy sessions, async Redis calls. No sync blocking calls inside async functions. |
| B2 | Never access model attributes not defined in the model. Check mapped columns before referencing them anywhere. |
| B3 | Use pytest for Python tests. Prefer test functions and fixtures, not `unittest.TestCase` or `unittest.main()`. |

## TypeScript / React Web Checks

These rules apply to projects using React and TypeScript. If the project uses a
monorepo layout (e.g. `apps/web`) or import aliases (`@/`), ensure they are correctly
configured and resolve to real files.

| # | Rule |
|---|---|
| F1 | Use the configured import aliases (e.g. `@/`) correctly instead of deep relative `../../` paths where applicable. |
| F2 | Conditional TanStack queries need an `enabled` guard. |
| F3 | Do not call hooks in the same component that renders `QueryClientProvider`; split provider and consumer. |
| F4 | Use TanStack Query for server state when that is the project's established pattern. |
| F5 | Components own UI only; data fetching belongs in custom hooks where the project uses that pattern. |
| F6 | `ApiClient` namespaces are object-literal class properties when that is the established local API shape. |
| F7 | Match TypeScript function signatures; use object params where declared. |
| F8 | Auth pages that hide the NavBar should use `min-h-screen`, not `min-h-[calc(100vh-4rem)]`. |
| F9 | After frontend refactors, grep for unused style constants, helper functions, and duplicate old local helper definitions. Vite/build output may not catch dead code or drift-prone duplication without linting. |
| F10 | After moving/removing/extracting React constants, helpers, style objects, or render functions, grep touched JSX for the moved names and related patterns such as `*Style`, `format*`, `render*`, and `*Label`; plain JavaScript Vite builds may not catch `ReferenceError` crashes in conditional render branches. |

## Mobile Checks

| # | Rule |
|---|---|
| M1 | In Expo / React Native apps, use relative imports unless the app explicitly configures an alias. |
| M2 | Verify export names against source before importing mobile components or hooks. |
| M3 | Do not `await` SecureStore in contexts that do not support async. |

## Infrastructure / CI Checks

| # | Rule |
|---|---|
| I1 | Verify the CI platform before adding pipeline config. Do not create GitHub Actions for a GitLab project or vice versa. |

## Codex-Specific Failure Patterns

### C1. Verify Text File Encoding After Generation

After generating plain text config, env, dependency, or manifest files, check encoding if
there is any risk the tool wrote UTF-16. Re-encode to UTF-8 before use.

Most dangerous files: `requirements.txt`, `.env*`, package manifests, CI config, and any
config parsed as plain text.

### C2. Treat Actual Code As Source Of Truth

When a plan, prompt, or memory disagrees with implementation, verify by reading the
actual endpoint, method, model, route, export, or schema definition before changing code.

### C3. Do Not Fake Imports In Tests

If imports fail because package initialization is broken, fix the real `__init__.py` or
module layout. Do not add workaround bootstraps such as `sys.modules["app.something"] = ...`
or `types.ModuleType(...)` in `conftest.py` unless the user explicitly requested a test-only shim.

### C4. Cross-Reference Env Vars Against Examples

When generating or changing `.env.example`, grep all env var references used by the app
and confirm each one appears in the example file. Check both frontend and backend env vars.

### C5. Retry Standards File Reads With Resolved Paths

If a standards file listed in `AGENTS.md` fails to read via a relative path, retry with a
resolved absolute path from the project root. Only report it missing if both attempts fail.
This guards against sandbox path translation issues around paths such as `../Dev-Standards/...`.

## Known Codex Risk Patterns

| Risk | What to check |
|---|---|
| Sandbox overconfidence | Code marked tested only ran in the sandbox. Verify imports, paths, env vars, and command context against the actual project. |
| Dependency drift | Packages installed in the sandbox may differ from manifests. Check `pyproject.toml`, `requirements.txt`, `package.json`, or lockfiles. |
| Path assumptions | Look for hardcoded absolute paths or relative paths that only work from Codex's working directory. |
| Intermediate artifacts | Remove scratch files, commented-out attempts, dead code, and TODO stubs. |
| Test environment bleed | Tests must not pass only because Codex seeded a sandbox DB. They should work against a fresh schema and fixtures. |
| File encoding | Verify generated text files are UTF-8 where relevant, especially dependency and config files. |
| Import workarounds | Search for fake modules or `sys.modules` injection added to avoid fixing real imports. |
| Env var drift | Cross-check new env vars against examples and docs. |

## Comparison Notes From Prior Reviews

| Dimension | Codex | Gemini |
|---|---|---|
| Missing files | Rare | Common |
| Logic bugs | Low | High, often several per phase |
| Over-engineering | Low | High |
| Encoding/artifact issues | UTF-16 file encoding observed | None observed |
| HTTP method accuracy | Usually reads actual code | May follow plan spec instead |
| Test structure | Often well-structured fixtures | Sometimes hardcoded IDs or improper mocking |
