# React / TypeScript Standards

Rules for React and React Native projects. Read `COMMON_CODING_STANDARDS.md` and
`JAVASCRIPT_TYPESCRIPT_STANDARDS.md` first — this file only covers rules specific to
React/JSX on top of those.

---

## Hooks Must Appear Before Any Early Return

Never place a `useState`, `useEffect`, `useMemo`, `useCallback`, or any other hook after
a conditional `return`. React requires hooks to be called in the same order on every
render; an early return that skips a hook causes a "Rendered fewer hooks than expected"
crash on the next route change. Move all hook calls to the top of the component, before
any guard clauses.

---

## No Stub Event Handlers

Every `onClick` / `onPress` handler must be fully implemented. If a feature is not ready,
disable the element (`disabled={true}`) or remove it from the JSX. Never ship a dead
interactive element — it confuses users and hides missing functionality.

---

## StrictMode `useEffect` Guard

Any `useEffect` that calls a mutating API (POST/PATCH/DELETE) must be guarded with a
`useRef(false)` flag set before the call and checked at the top:

```js
const hasFired = useRef(false);
useEffect(() => {
  if (hasFired.current) return;
  hasFired.current = true;
  // ... mutating call
}, []);
```

React 18 StrictMode double-invokes effects in development, so an unguarded mutating
effect will fire twice and produce duplicate writes.

---

## JSX Tag Casing

Match opening and closing tag names exactly, including casing.
A mismatch (`<MyComponent>` / `</mycomponent>`) causes a runtime crash with a misleading
error. Triple-check casing on all closing tags after any rename.

---

## No Portals or Notification Providers Inside `ErrorBoundary`

`<Toaster />`, modal roots, and similar portal elements must be siblings of the
`ErrorBoundary`, not children inside it. When the boundary catches an error and renders
its fallback, any portals inside it are unmounted, causing toast notifications to
disappear exactly when error feedback is most needed.

---

## `setInterval` / `setTimeout` Ref Pattern

Any interval or timeout stored in a ref must be:
1. Cleared before being reassigned (in event handlers that could fire multiple times).
2. Cleaned up in the `useEffect` return.
3. Guarded against "component unmounted during async" races when scheduled inside a
   `.then()` / async callback.

This applies to every reassignment site, including click handlers, polling starters,
and async continuations after `await`. If code writes `timerRef.current = setInterval(...)`
or `timerRef.current = setTimeout(...)`, there should normally be a
`clearInterval(timerRef.current)` or `clearTimeout(timerRef.current)` immediately before
that assignment unless a nearby comment explains why overlap is intentional.

```js
useEffect(() => {
  let cancelled = false;
  fetchStatus().then((s) => {
    if (cancelled) return;
    if (s.running) {
      pollRef.current = setInterval(pollStatus, 1500);
    }
  });
  return () => {
    cancelled = true;
    clearInterval(pollRef.current);
  };
}, []);

function handleRunNow() {
  clearInterval(pollRef.current);   // clear existing before reassigning
  pollRef.current = setInterval(pollStatus, 1500);
}
```

---

## Shared Helper Extraction

After extracting React helpers, constants, labels, formatters, or parsers into a shared
module, grep for the old local definitions and either remove them or document why the
behavior must intentionally differ. Build tools often do not catch duplicate helper
drift, and duplicated UI constants such as status labels, skip reasons, source/platform
labels, URL normalizers, and parser regexes can silently diverge across panels.

---

## Refactor Undefined Identifier Checks

After moving, removing, or extracting React component-local constants, helpers, style
objects, or render functions, grep the touched component for the moved identifier names
and for related naming patterns such as `*Style`, `format*`, `render*`, and `*Label`.
Every JSX reference must still resolve in the same module scope or through an explicit
import.

Do not rely on a Vite production build alone to catch this. Plain JavaScript React
projects can build successfully while a conditional render path still throws
`ReferenceError: <name> is not defined` only when that branch is opened. If the project
does not have ESLint or TypeScript `no-undef` coverage, manually inspect the touched JSX
branches and run the closest available grep/check before finalizing.

---

## Verify JSX Edits With a Build

After editing any `.jsx`/`.tsx` file, run the project's production build (e.g.
`vite build`, `next build`) before declaring the change done — not just a linter.

JSX syntax errors that a build catches instantly can otherwise ship silently:

- `{/* comment */}` is only valid as a JSX *child*. Placed between attributes in an
  opening tag (e.g. `<Foo a={1} {/* note */} b={2}>`), it is a syntax error that breaks
  the entire module's render with no runtime warning — the component (and often its
  whole parent tree) just stops rendering.
- Mismatched braces/parens introduced mid-edit in a large attribute list.

If `eslint` fails to run (missing config, wrong version, etc.), do not treat that as
"good enough" — fall back to the build, which requires no extra config and catches these
errors in seconds. Use eslint as an *additional* check when it works, not a replacement
for the build.

---

## TanStack Query

- **`enabled` guard on conditional queries** — any `useQuery` that should only fire for
  certain roles or states must pass an explicit `enabled: <boolean expression>`. Don't
  rely on error handling to suppress requests that shouldn't be made at all.
- **`QueryClientProvider` placement** — never call `useQuery`-based hooks in the same
  component that renders `QueryClientProvider`. Split into an outer provider component
  and an inner consumer.

---

## React Native / Mobile

- **Relative imports only** — do not use the `@/` alias in `apps/mobile`. Use `../`
  relative paths. The alias is not configured in the mobile app.
- **Verify export names against source** — do not assume mobile component/hook export
  names match the web app. Always check the actual source file.
- **SecureStore calls are synchronous-style** — do not `await` SecureStore in contexts
  that don't support async (e.g. synchronous initializers).
