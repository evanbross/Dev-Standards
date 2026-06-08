# Codex Session Rules

Read this at session start. Keep it short; detailed review checks live in
`CODEX_REVIEW_CHECKLIST.md` and should be loaded before finalizing code edits.

## Required Behavior

Rules are numbered for stable reference in review notes. Do not renumber when adding.

| # | Rule |
|---|---|
| G0 | Ask once before editing files when the project requires confirmation. Once approved, continue without asking again unless there is unusual risk or a real blocker. |
| G1 | Follow the requested file layout exactly: no extra directories, parallel implementations, or bonus files. |
| G2 | Re-read the standards listed in the project's `AGENTS.md` profile at session start. |
| G3 | Do not revert user changes unless explicitly asked. Work with unrelated dirty files. |
| G4 | Use the actual codebase as source of truth when a plan or memory disagrees with implementation. |
| G5 | Prefer narrow edits that match existing local patterns. |
| G6 | AI attribution (in-code `# ai:` comments and `AI-Model:` git trailers): see `COMMON_CODING_STANDARDS.md` § AI Attribution. |

## High-Risk Codex Habits To Prevent

- Verify real endpoint, method, model, and export definitions before using them.
- Do not inject fake modules into `sys.modules` to work around broken imports; fix the actual import/package issue.
- When generating config, dependency, or env files, verify they are UTF-8 if tooling may have created UTF-16.
- Cross-check env var references against `.env.example` when touching env config.
- Do not claim sandbox-only verification proves behavior in the user's real environment.

## Language Defaults

- Python tests use pytest-style test functions and fixtures.
- Do not introduce `unittest.TestCase` or `unittest.main()` unless converting the whole touched file is out of scope.
- For React, hooks must stay before early returns and interval refs must be cleared before reassignment and on cleanup.
