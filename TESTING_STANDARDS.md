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

## Extract Pure Logic Out of Entry-Point Modules

Entry-point files (`app.py`, `main.py`) often run side effects at import time — DB
initialization, background threads, starting a dev server — which makes importing the
module directly in a test slow, order-dependent, or unsafe. If you need to unit-test a
decision made inside an entry-point hook (a `before_request`/middleware guard, a startup
check), extract the pure decision logic into a plain function in a shared module and have
the entry point call it. Test the extracted function directly with plain arguments; don't
try to import the entry-point module just to reach it.
