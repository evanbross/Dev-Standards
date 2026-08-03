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

## No Mocked Databases in Integration Tests

Integration tests must run against a real database instance, not mocks. A test suite that
passes against mocks but fails against the real schema provides false confidence — the
mock diverges from the real schema over time and the divergence goes undetected until
production.

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
