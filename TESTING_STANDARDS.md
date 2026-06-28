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
