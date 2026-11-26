# Testing Standards

Rules for writing and maintaining automated tests. Read `COMMON_CODING_STANDARDS.md` first.

---

## Test Framework — Always Use pytest

Use **pytest** for all Python tests. Do not use `unittest` or its `TestCase` classes.

- Test functions, not classes: `def test_something():` not `class TestSomething(unittest.TestCase):`
- Plain `assert` statements: `assert result == 42` not `self.assertEqual(result, 42)`
- Exception assertions: `pytest.raises(ValueError, match="...")` not `assertRaisesRegex`
- Mocks: `unittest.mock.patch` is fine — it works with pytest and there is no need to replace it
- No `if __name__ == "__main__": unittest.main()` block

`unittest`-style tests in the codebase are technical debt. Convert them to pytest when you touch the file.

---

## Scope pytest Collection with `testpaths`

Always set `testpaths` in `pyproject.toml` (`[tool.pytest.ini_options]`) or `pytest.ini`
so the bare `pytest` command — with no path argument — only collects the actual test
directory. Without it, pytest's default rootdir collection walks the entire project,
including scratch/output directories (`tmp/`, build artifacts, `.venv`) that were never
meant to be collected and can fail for reasons unrelated to the test suite (permission
errors, syntax in a stray script, etc.).

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
norecursedirs = ["tmp", ".venv", "frontend/dist", "node_modules"]
```

**Why:** contributors and CI both reach for the bare `pytest` command by habit. A failure
unrelated to the tests themselves gets misread as "the suite is broken," burying real
regressions in the noise.

---

## Run Targeted Tests While Iterating; the Full Suite Once

**This applies to every code edit, not only edits to test files.** Size verification to
the change:

- **While iterating:** run only the test files that exercise the code you touched, e.g.
  `uv run pytest -q tests/test_contacts_routes.py`. Find them with
  `grep -rl "<module or function name>" tests/`.
- **Changes no test exercises** (string/typo fixes, imports, startup wiring, config,
  comments): run the narrowest direct check instead — an import (`python -c "import app"`),
  the linter on the touched files, or the one command that exercises the change.
- **Full suite:** once before committing, or leave it to CI. Never after every edit.

A project whose full suite takes more than a few minutes should say how long in its
`CLAUDE.md`/`AGENTS.md`, so agents know the cost of reaching for it.

**Why:** a 15-minute suite run after each of several small edits turns a typo fix into a
half-hour task. Targeted runs catch the same regressions in the code that changed; the
pre-commit full run (or CI) catches the rest.

---

## No Mocked Databases in Integration Tests

Integration tests must run against a real database instance, not mocks. A test suite that
passes against mocks but fails against the real schema provides false confidence — the
mock diverges from the real schema over time and the divergence goes undetected until
production.

### New or Changed Queries Must Execute Against the Real Schema

Every new or materially changed repository query, view/function call, ORM mapping, or
migration-dependent read/write path must have an integration test that executes it
against a freshly initialized real database schema. Assert the returned shape and the
important semantics, not merely that the call did not raise.

A route or service test with a fake repository does not satisfy this requirement: it
tests orchestration while bypassing the SQL and schema contract. Static compilation,
linters, and type checkers also do not validate embedded SQL identifiers. Keep those
checks, but pair them with the database integration test that can detect missing columns,
incorrect aliases, invalid joins, constraint mismatches, and provider-specific syntax.

When a production query fails because of schema drift or an invalid assumption, add the
regression test to the repository/data-access layer and audit sibling queries that use
the same table, view, function, alias, or assumption.

---

## Test Fixtures Must Match the Actual Response Schema

Before writing a fixture that reads a field from a response, verify that field exists in
the declared `response_model` or actual return value. Do not assume nested data is
present without checking the schema. Reading a field that doesn't exist produces `None`
silently in some frameworks and `KeyError` in others.

---

## Tests Must Work Against a Fresh Schema

Do not seed the test database inside a sandbox and leave tests hardcoded against that
state. Use fixtures and `conftest.py` (or equivalent) so the test suite works correctly
against a freshly-migrated, empty schema.

---

## Extract Pure Logic Out of Entry-Point Modules

Entry-point files (`app.py`, `main.py`) often run side effects at import time — DB
initialization, background threads, starting a dev server — which makes importing the
module directly in a test slow, order-dependent, or unsafe. If you need to unit-test a
decision made inside an entry-point hook (a `before_request`/middleware guard, a startup
check), extract the pure decision logic into a plain function in a shared module and have
the entry point call it. Test the extracted function directly with plain arguments; don't
try to import the entry-point module just to reach it.

---

## Keep Test Fakes in Sync with Real Interfaces

When a test uses a fake/stub class (stand-in for a real repository, service, or client)
and the real class's constructor signature or method interface changes, all fakes must be
updated in the same commit.

**Repository fakes:** If a route gains a new `repos["x"]` call, every test that
monkeypatches `get_repos` for that route must add `"x"` to its fake repos dict. A
missing key causes the route to raise `KeyError` before the behavior under test is
reached, producing a misleading test failure.

**Constructor fakes:** If a real class gains new constructor arguments (including keyword
arguments), any fake that substitutes for it must accept those arguments. The minimal
fix is `def __init__(self, *args, **kwargs): pass` when the fake doesn't use them.

**Checklist when changing a route:**
1. If you add `repos["x"]` to a route, grep test files for the route's blueprint name
   and add `"x"` to every matching fake repos dict.
2. If you change a constructor signature that is monkeypatched in tests, grep for the
   class name in the test directory and update any fakes.

---

## Failure Ordering and Uncertain Delivery

### Out-of-Order Completion

Any code path that can have more than one async/network call in flight for the same
logical key (a revision-triggered refetch while an older fetch is still pending, a
double-click, two retried requests racing) must have a test where the responses settle
**out of order** — the older request resolving last. Assert that a stale response cannot
overwrite state a newer response already set, on both the success path and the error
path. See `REACT_STANDARDS.md` § Guard Async Completions By Request Identity for the
pattern this tests.

### A Database Write Plus a Separate External Side Effect

Any path that performs a DB write (claiming a task, marking a day's digest as sent) and
then a separate, uncertain external side effect (sending an email, calling a third-party
API) must have a test for the failure-between-the-two case: the external call raises, or
the process exits, after the DB write commits but before the side effect completes.

- Require the code under test to have an explicit, stated policy for this case — a
  bounded retry for known-transient failures, or an explicit "claimed but undelivered"
  state a reconciliation pass can detect — not silence. A test that only exercises "the
  happy path, then the already-claimed path" does not cover this.
- A retry must not be allowed to duplicate a side effect that already happened; if the
  external outcome is ambiguous (e.g. the send timed out but may have gone through),
  the test should assert the code's documented choice for that ambiguity rather than
  assuming at-most-once or at-least-once by default.

### Migrations: Empty Init and a Representative Upgrade

A migration test suite must cover both: initializing a fresh, empty schema, **and**
upgrading from a representative prior-version schema populated with data. Run both using
the same database role the running application actually connects as — not a superuser —
so a privilege gap (missing grant on a new table/sequence) surfaces in the test instead of
in production.
