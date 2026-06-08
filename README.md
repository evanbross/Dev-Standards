# Dev Standards

Shared coding standards for all AI coding assistants across all projects.

---

## Structure

Standards are split by concern so each project only loads the files relevant to its stack.
Each AI assistant has its own file that lists which standards apply and adds AI-specific behavior rules.

### Universal

| File | Contents |
|------|----------|
| `COMMON_CODING_STANDARDS.md` | Session startup, SOLID/security, dependency tracking, **AI attribution rule (single source of truth)** |

### By Language / Framework

| File | Contents |
|------|----------|
| `PYTHON_STANDARDS.md` | Imports, naming, datetime, exceptions, service layer log-and-re-raise, stubs, dead code, transactions, upserts |
| `FASTAPI_STANDARDS.md` | FastAPI/OpenAPI completeness, SQLAlchemy 2.x, async rules |
| `REACT_STANDARDS.md` | TypeScript, React hooks, JSX, StrictMode, React Native / mobile |
| `DATABASE_STANDARDS.md` | Repository pattern, database abstraction, Alembic migrations, GeoAlchemy2 |
| `TESTING_STANDARDS.md` | Integration test DB, fixture schema accuracy, fresh-schema requirement |
| `INFRASTRUCTURE_STANDARDS.md` | Dockerfiles, CI platform selection, plan file layout |
| `GOOGLE_CODING_STANDARDS.md` | Google Sheets/Drive/Gmail/Workspace APIs — auth, HttpError, dynamic ranges, fields, colors, scopes |
| `WINDOWS_STANDARDS.md` | Line endings (CRLF), pywin32 services, NSSM |
| `JAVA_CODING_STANDARDS.md` | Java projects |

### By AI Assistant

| File | Contents |
|------|----------|
| `CLAUDE.md` | Claude Code preferences, response style, project profiles |
| `CODEX_CODING_STANDARDS.md` | Codex behavior rules, observed failure modes, risk checklist |
| `GEMINI_SESSION_STANDARDS.md` | Gemini behavior rules, observed failure modes |
| `QWEN_CODING_STANDARDS.md` | Qwen behavior rules (failure modes added as observed) |

---

## Project Profiles

Each project's `AGENTS.md` (or equivalent agent config) lists which standards files apply.
This keeps AI prompts focused — a Python-only backend doesn't load React rules.

**Example — Full-stack app:**
```
COMMON_CODING_STANDARDS.md
PYTHON_STANDARDS.md
FASTAPI_STANDARDS.md
DATABASE_STANDARDS.md
REACT_STANDARDS.md
TESTING_STANDARDS.md
INFRASTRUCTURE_STANDARDS.md
```

**Example — Python app with Google Sheets integration:**
```
COMMON_CODING_STANDARDS.md
PYTHON_STANDARDS.md
DATABASE_STANDARDS.md
REACT_STANDARDS.md
GOOGLE_CODING_STANDARDS.md
TESTING_STANDARDS.md
```

**Example — Windows automation script:**
```
COMMON_CODING_STANDARDS.md
PYTHON_STANDARDS.md
WINDOWS_STANDARDS.md
```

---

## Adding Rules

1. Add the rule to the appropriate standards file.
2. If it's AI-specific (a failure mode or behavior pattern unique to one assistant), add it to that assistant's file instead.
3. If it's truly universal, add it to `COMMON_CODING_STANDARDS.md`.
4. Do not renumber existing rules — numbers are stable review references.

**Never duplicate a rule from `COMMON_CODING_STANDARDS.md` into an AI-specific file.**
AI-specific files should reference COMMON rules by name/section, not copy them.
The AI attribution rule is a specific example: it lives only in `COMMON_CODING_STANDARDS.md`
§ AI Attribution. Every AI-specific file has a one-line pointer to it; updating
COMMON automatically applies to all assistants.

---

## Adding a New AI Assistant

1. Create `<AI>_SESSION_RULES.md` (or equivalent) in this directory.
2. Include the standard boilerplate at minimum:

```markdown
# <AI Name> Coding Standards

## Standards Files — Read These First

At the start of every session, re-read this file and every standards file listed in the
project's agent config before making any changes.

## General Behavior Rules

| # | Rule |
|---|---|
| G0 | **Re-read standards at session start** — see `COMMON_CODING_STANDARDS.md`. |
| G1 | **Follow the file layout in the plan exactly** — no extra directories, no parallel implementations, no bonus files. |
| G2 | **AI attribution** (in-code `# ai:` comments and `AI-Model:` git trailers): see `COMMON_CODING_STANDARDS.md` § AI Attribution. |

## <AI>-Specific Behavior Patterns

*No failure modes observed yet. Add entries here as patterns emerge.*
```

3. Add the file to the "By AI Assistant" table in this README.
4. Reference it in each project's `AGENTS.md` that will use this assistant.
