# JavaScript / TypeScript Standards

Rules for all JavaScript and TypeScript code — Node scripts, browser extensions,
userscripts, build tooling, and any other non-React JS/TS. Read
`COMMON_CODING_STANDARDS.md` first. React and React Native projects also read
`REACT_STANDARDS.md`, which covers JSX/hooks-specific rules on top of these.

---

## TypeScript

- **No `any` types** — all values must be typed. `any` defeats the type checker for every
  consumer of that value downstream.
- **No `// @ts-ignore`** — fix the underlying type error instead of suppressing it.
- **Shared types centrally** — never redeclare API response types or request body shapes
  locally in a component or hook. All shared types belong in a shared types package and
  must be imported from there.

---

## Derived Values Must Be Declared After Their Dependencies

`const` declarations are not hoisted with a value — referencing a `const` before its
declaration line throws `ReferenceError: Cannot access 'X' before initialization` (the
temporal dead zone). This shows up as a cryptic minified-variable error in production
builds (e.g. `Cannot access 'm' before initialization`).

Always declare computed values after all the variables they depend on:

```js
// Wrong — loginInProgress references running before running is declared:
const loginInProgress = running && status?.message?.includes("...");
const running = !!status?.running;

// Correct — running is declared first:
const running = !!status?.running;
const loginInProgress = running && status?.message?.includes("...");
```

---

## Write Calls Must Check `res.ok`

After any `fetch` that writes data (POST/PUT/DELETE), check `res.ok` before refetching
or updating state. Silently refetching on a 4xx/5xx produces stale data with no
user-visible error.

```js
const res = await fetch("/api/...", { method: "PUT", ... });
if (!res.ok) {
  window.alert(`Save failed: ${res.status}`);
  return;
}
await refetch();
```

---

## Numeric Inputs Must Be Validated Before Sending

`parseInt` / `parseFloat` return `NaN` for empty or non-numeric input. `NaN` serialises
to `null` in JSON. Validate before sending:

```js
const days = parseInt(value, 10);
if (!Number.isFinite(days) || days < 1) {
  window.alert("Enter a valid number.");
  return;
}
```

---

## `window.confirm` — Use the Qualified Form

Always write `window.confirm(...)`, never bare `confirm(...)`. The qualified form is
unambiguous to linters and consistent with how all other global browser APIs are called.
