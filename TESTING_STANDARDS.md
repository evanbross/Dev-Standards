# Testing Standards

Rules for writing and maintaining automated tests. Read `COMMON_CODING_STANDARDS.md` first.

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
