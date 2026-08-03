# Java Coding Standards

## Naming & Conventions

- `camelCase` for methods and variables, `PascalCase` for classes, `UPPER_SNAKE_CASE` for constants
- Boolean methods use an `is`, `has`, or `can` prefix (`isConnected()`, not `connected()`)
- No single-letter names except loop counters (`i`, `j`) — even short-lived lambda parameters deserve readable names

## Nullability

- Annotate all public method parameters and return types with `@NotNull` or `@Nullable`
- Never return `null` from a method with a `Collection`, `List`, or `Map` return type — return an empty collection instead
- Use `Optional<T>` only when the caller genuinely needs to distinguish "not found" from other states; don't use it as a general null-avoidance wrapper on every method

## Immutability & `final`

- Declare all instance fields `final` unless mutation is genuinely required
- Use `List.of()` / `Map.of()` for fixed data; when a mutable copy is needed, take it explicitly (`new ArrayList<>(source)`)
- Use `record` for any class that is purely a data carrier with no behavior beyond accessors

## Exception Handling

- Never swallow exceptions silently — a `catch` block with no logging or rethrow is always wrong
- Public API methods should declare specific checked exceptions rather than bare `throws Exception`; the broad signature forces callers to handle programming errors alongside legitimate failures
- Don't use exceptions for control flow (e.g., catching a parse error to detect a bad input format)

## Resource Management

- Use try-with-resources for everything implementing `AutoCloseable` — streams, readers, connections
- `HttpClient` is thread-safe and heavyweight; construct it once as a class field or application-level singleton rather than per-call

## Thread Discipline

- All Swing/UI mutations must happen on the EDT — use `ApplicationManager.getApplication().invokeLater(...)`
- All blocking work (network, disk, long computation) must run on a pooled thread — `executeOnPooledThread(...)` or equivalent
- If a field is only safe to access from one thread, document that constraint with a comment

## Code Organization

- Annotate every overriding method with `@Override` — the compiler then catches stale overrides before they silently become no-ops
- Return and accept interface types (`List<T>`, not `ArrayList<T>`) — callers should not depend on the concrete implementation
- Use section-comment dividers (e.g. `// ── UI construction ──`) to group logical blocks within larger classes

## Modern Java Idioms (Java 17+)

- Use `var` for local variables where the type is obvious from the right-hand side — reduces noise without losing clarity
- Prefer switch expressions over switch statements for assignments; they enforce exhaustiveness
- Use `instanceof` pattern matching rather than a cast-after-check:
  ```java
  // prefer
  if (obj instanceof String s) { ... }
  // over
  if (obj instanceof String) { String s = (String) obj; ... }
  ```

## Testing

- Test method names describe behavior, not the method under test:
  `chat_throwsOnNon200Response()` not `testChat()`
- Structure each test as Arrange / Act / Assert with a blank line separating each section
- Mock at the boundary (HTTP responses, file I/O) — don't mock internal collaborators you own
