# Global Claude Code Preferences

## Response Style

Aim for a mid-ground level of explanation. Don't be terse — explain *why* decisions are made, not just *what* to do. This helps with learning. Avoid excessive verbosity or padding, but don't skip reasoning when it's useful.

## Session Startup

At the start of every conversation, re-read the standards files listed in the project's agent config (e.g. `AGENTS.md`) before making changes. This CLAUDE.md is auto-loaded by the system, but the per-language and per-framework standards files are not — rules may have been added to them since the last session.

## Code Review

After writing or modifying code, proactively flag any violations of the best practices defined in the standards files. Don't wait to be asked — a brief "a few things worth noting:" section at the end of a response is appropriate.

## Project CLAUDE.md Propagation

When starting work in any project that has a `CLAUDE.md` file, check whether it references
the appropriate standards files from `../Dev-Standards/` via `@` includes. If any are
missing for the project's stack, add them before doing other work.

The pattern for every project `CLAUDE.md` is a list of `@` includes:

```
@../Dev-Standards/COMMON_CODING_STANDARDS.md
@../Dev-Standards/PYTHON_STANDARDS.md
...
```

### Always include
- `COMMON_CODING_STANDARDS.md` — universal rules; every project needs this

### Include when the stack applies

| File | When to include |
|------|-----------------|
| `PYTHON_STANDARDS.md` | Any Python code in the project |
| `DATABASE_STANDARDS.md` | Any project with a database (SQL, Redis, etc.) |
| `JAVASCRIPT_TYPESCRIPT_STANDARDS.md` | Any JavaScript or TypeScript code (Node scripts, extensions, userscripts) |
| `REACT_STANDARDS.md` | React or React Native frontend (alongside `JAVASCRIPT_TYPESCRIPT_STANDARDS.md`) |
| `FASTAPI_STANDARDS.md` | FastAPI projects (alongside `PYTHON_STANDARDS.md`) |
| `TESTING_STANDARDS.md` | Any project with automated tests |
| `INFRASTRUCTURE_STANDARDS.md` | Projects with Docker, CI/CD pipelines, or deployment config |
| `GOOGLE_CODING_STANDARDS.md` | Any use of Google Sheets, Drive, Gmail, Calendar, or Workspace APIs |
| `WINDOWS_STANDARDS.md` | Windows-specific tooling or PowerShell automation |
| `JAVA_CODING_STANDARDS.md` | Java projects |

### Removing stale references
If a standards file is referenced but no longer applies to the project (e.g. the project
dropped its React frontend, or removed all Google API usage), remove the `@` include.
Stale references waste context and can produce irrelevant lint feedback.

### Do not include in CLAUDE.md
AI-agent-specific files (`CODEX_CODING_STANDARDS.md`, `GEMINI_CODING_STANDARDS.md`,
`QWEN_CODING_STANDARDS.md`) belong in the project's `AGENTS.md`, not `CLAUDE.md`.
Each agent file is written for that agent's quirks and is not useful context for Claude.

## Code Generation Behavior

These are Claude-specific behavioral rules that go beyond what's in the coding standards files:

- **When generating multiple files in one session** — pause between files and flag uncertainty rather than patching in place. A `# TODO: verify this pattern` comment is better than a working-looking hack.

- **No `#` comments inside quoted Bash `-c` strings** — a newline followed by `#` inside a quoted argument triggers Claude Code's path-validation warning ("Newline followed by # inside a quoted argument can hide arguments from path validation"). Never put comment lines inside `python -c "..."` or similar inline scripts. If the script needs comments or is longer than ~2 lines, write it to a temp file and run that instead.

- **AI attribution** — In-code `# ai:` comments and `AI-Model:` git trailers are required. Full rule: `COMMON_CODING_STANDARDS.md` § AI Attribution.

- **Complete wiring steps immediately** — if you notice a gap between what was implemented and what needs to be connected (a prop added but not passed, a handler written but not called), close it in the same turn. Never leave it as a "pending task" for the next session.

## Context Window Management

Proactively suggest a `/clear` or `/compact` when the conversation is getting long — specifically:
- After completing a multi-step implementation block (e.g., finishing a numbered phase step)
- When 5+ large files have been read in the session
- Before starting a new major phase or feature

Mention it briefly at the end of a response: e.g., "This is a good point to `/clear` before continuing with Step 4."

## Environment

- Platform: Windows 10
- Primary shell: bash (use Unix syntax — forward slashes, `/dev/null`, etc.)
- Common stack: Python (Flask), React (Vite), PostgreSQL/SQLite
