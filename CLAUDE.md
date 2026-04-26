# Global Claude Code Preferences

## Response Style

Aim for a mid-ground level of explanation. Don't be terse — explain *why* decisions are made, not just *what* to do. This helps with learning. Avoid excessive verbosity or padding, but don't skip reasoning when it's useful.

## Session Startup

At the start of every conversation, re-read the standards files listed in the project's agent config (e.g. `AGENTS.md`) before making changes. This CLAUDE.md is auto-loaded by the system, but the per-language and per-framework standards files are not — rules may have been added to them since the last session.

## Code Review

After writing or modifying code, proactively flag any violations of the best practices defined in the standards files. Don't wait to be asked — a brief "a few things worth noting:" section at the end of a response is appropriate.

## Project CLAUDE.md Propagation

When starting work in any project that has a `CLAUDE.md` file, check whether it contains each of the following sections. If any are missing, add them before doing any other work.

Sections that must be present in every project CLAUDE.md:
- **Best Practices** — language/framework rules, SOLID, secure coding, idiomatic patterns
- **Database Abstraction** — repository pattern, dependency-injected sessions, no provider leakage, abstract base/Protocol per repository

## Code Generation Behavior

These are Claude-specific behavioral rules that go beyond what's in the coding standards files:

- **When generating multiple files in one session** — pause between files and flag uncertainty rather than patching in place. A `# TODO: verify this pattern` comment is better than a working-looking hack.

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
