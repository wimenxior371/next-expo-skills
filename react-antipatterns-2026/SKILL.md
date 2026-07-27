---
name: react-antipatterns-2026
description: Detect and fix the eight patterns that quietly degrade React codebases in the React 19 / React Compiler era — gratuitous memoization, useEffect misuse, mixed concerns, index keys, state that belongs in the URL, and business logic tangled with React. Use when writing new React components, reviewing or refactoring React code, or auditing a codebase for maintainability.
---

# react-antipatterns-2026

Eight underlying habits that produce essentially every codebase smell in modern React (19.x, 2026), and the fix for each.

Key context: React Compiler v1.0 (stable Oct 2025, default in Next.js 16) and `useEffectEvent` (React 19.2) have **inverted** some historical advice — notably around memoization. Advice from 2020–2023 ("wrap everything in `useCallback`") is now often actively wrong.

## When to use

- Writing or reviewing React / Next.js / Expo (React Native) component code.
- Refactoring a legacy codebase to modern (React 19+, Compiler-enabled) patterns.
- Deciding whether to add a dependency, a component, a hook, or a `useEffect`.

## The eight patterns

### 1. Reaching for libraries when the language already does the job

The reflex to install a package for every problem accumulates dependencies the team neither understands nor monitors — each a security-audit obligation, upgrade hazard, and cognitive tax on every new contributor.

**Test before adding any utility package:** can two lines of native JavaScript do this? If yes, don't add it.

```js
// Lodash → Native
_.isEmpty(arr)         → arr.length === 0
_.isNil(x)             → x == null
_.cloneDeep(obj)       → structuredClone(obj)
_.get(obj, 'a.b.c')    → obj?.a?.b?.c
_.uniq(arr)            → [...new Set(arr)]
_.groupBy(arr, fn)     → Object.groupBy(arr, fn)   // ES2024
// Moment/dayjs → Native (Temporal stabilizing)
moment().format('YYYY-MM-DD') → new Date().toISOString().slice(0, 10)
// classnames → Native
classnames({ active: isActive }) → `${isActive ? 'active' : ''}`
```

### 2. Writing 2022-era memoization when the React Compiler exists

React Compiler v1.0 inserts memoization automatically at build time — better and more selectively than hand-written `useMemo` / `useCallback` / `React.memo`. Manual memoization in a Compiler-enabled codebase is counterproductive noise.

```jsx
// ❌ 2022 pattern — no longer correct with the compiler
const TodoList = React.memo(({ todos, onToggle }) => {
  const sortedTodos = useMemo(() => [...todos].sort((a, b) => a.priority - b.priority), [todos]);
  const handleToggle = useCallback((id) => onToggle(id), [onToggle]);
  ...
});

// ✅ 2026 pattern — simplest possible component, trust the compiler
function TodoList({ todos, onToggle }) {
  const sortedTodos = [...todos].sort((a, b) => a.priority - b.priority);
  return (
    <ul>
      {sortedTodos.map(todo => <TodoItem key={todo.id} todo={todo} onToggle={onToggle} />)}
    </ul>
  );
}
```

Rules:
- New projects: enable the compiler from day one; never write `useMemo`/`useCallback`/`React.memo` unless **profiling** shows a specific need the compiler missed.
- Legacy codebases: removing existing memoization usually helps more than adding it.
- Exception: if the codebase genuinely cannot adopt the compiler, historical memoization rules still apply — but the smell to look for is *gratuitous* memoization, not missing memoization.

### 3. Using `useEffect` for things that are not effects

The single largest source of React bugs. An effect exists to **synchronize with an external system** (browser API, subscription, non-React widget). If the job is "calculate something" or "respond to a user action," the answer is almost never `useEffect`. (See react.dev: "You Might Not Need an Effect".)

**Deriving state from props** — calculate during render instead:

```jsx
// ❌ effect mirroring props into state
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(`${user.firstName} ${user.lastName}`); }, [user.firstName, user.lastName]);

// ✅ calculate during render
const fullName = `${user.firstName} ${user.lastName}`;
```

**Resetting state when props change** — use a `key` to remount instead:

```jsx
// ❌ useEffect(() => { setDraft(''); }, [userId]);
// ✅
<EditUserForm key={userId} userId={userId} />
```

**Handling user events** — put the logic in the event handler, not in an effect triggered by a state flag:

```jsx
// ❌ setSubmitted(true) in onClick + effect watching `submitted`
// ✅
const handleCheckout = async () => {
  await fetch('/api/checkout', { method: 'POST', body: JSON.stringify(items) });
};
```

For genuinely needed effects with stale-closure problems, use React 19.2's `useEffectEvent` — effect logic that always sees the latest props/state without adding them to the dependency array.

### 4. Treating components as if they're free

Every component boundary is a prop interface, a type signature, a re-render boundary, a debugging unit, and a file every reader must open and hold in working memory. A feature spread across 40 tiny files is harder to understand than one readable block.

Heuristic: **a component should exist when something is reused or has measurable cognitive complexity — not by default.** A 40-line JSX block used once is clearer as a 40-line block than as five 8-line components.

- Same for custom hooks: a hook wrapping a single `useState` isn't earning its complexity; a hook coordinating state + effects + subscriptions usually is.
- Smell: wrappers with no logic, no state, no props beyond `children`, used in exactly one place — "a comment in disguise." Delete and inline the JSX.

### 5. Mixing concerns within components

A component that fetches data, manages loading/error state, and renders UI is doing four jobs, none well. The converged 2026 architecture:

- **Server state** (API data, cached, revalidated): TanStack Query / SWR / RTK Query — never `useState` + `useEffect` + `fetch`.
- **Client UI state** (modals, forms, filters, theme): Zustand for cross-component; local `useState` for component-scoped.
- **Server-rendered data**: React Server Components (Next.js) — fetching never reaches the client.

```jsx
// ❌ hand-rolled: useState ×3 + useEffect + cancellation flag (~25 lines)
// ✅ TanStack Query — plus dedup, caching, refetch, retry, cancellation for free
function UserProfile({ id }) {
  const { data: user, isPending, error } = useQuery({
    queryKey: ['user', id],
    queryFn: () => fetch(`/api/users/${id}`).then(r => r.json()),
  });
  if (isPending) return <Spinner />;
  if (error) return <Error error={error} />;
  return <Profile user={user} />;
}
```

Hand-rolling server state in 2026 is an active mistake, not a stylistic choice.

### 6. Index keys, mutation, and other shape-of-data errors

**Index keys on dynamic lists** — when an item is deleted/reordered, React binds local state (inputs, focus, animations) to the wrong items:

```jsx
// ❌ key={i}    ✅ key={todo.id}  — stable, data-derived
```

**Mutating state directly** — React uses referential equality; a mutated array is the same reference, so no re-render ("my UI isn't updating"):

```jsx
// ❌ items.push(item); setItems(items);
// ✅ setItems([...items, item]);
// ✅ or Immer / Zustand's Immer middleware for complex updates
```

**Stale closures in handlers** — any value read inside an effect/callback must be in the dependency array or accessed via a latest-value mechanism (`useEffectEvent` in 19.2 replaces the old `useRef` gymnastics).

Keep the lint rules that catch these enabled.

### 7. Confusing URL state with component state

The URL is application state that survives reload, can be shared, bookmarked, and back-buttoned. Storing filters, pagination, search queries, sort order, or view-mode toggles in `useState` throws all of that away — the shared link shows the default view.

- **URL** (Next.js `searchParams`, React Router `useSearchParams`, TanStack Router): anything the user might bookmark, share, or restore.
- **Local state**: genuinely ephemeral things — open dropdowns, half-typed input, hover.

### 8. No clear seam between business logic and React

React is a UI library. Domain logic (pricing, tax rules, formatting records) belongs in plain TypeScript modules that **never import from `react`**. Components import from domain logic — never the reverse.

Why the seam pays:
- Pure functions test without rendering, a DOM, or `@testing-library/react`.
- Domain logic outlives React versions and moves between contexts (Node, workers, React Native) freely.
- Contributors can reason about domain and UI separately.

Smell: `formatCurrency` living inside a component file; business rules embedded in JSX. Extracting them is one of the highest-leverage refactors available.

## The underlying principle

These aren't React problems — they're general engineering problems (coupling, premature abstraction, mixed concerns, ignoring existing tools) showing up in React code. Keep the surface area small. Let the platform do its job. Separate concerns. Don't solve problems you don't have.

## Instructions

When writing or reviewing React code, check against all eight patterns:

1. **Dependencies**: before adding a utility package, apply the two-lines-of-native-JS test.
2. **Memoization**: if the React Compiler is (or could be) enabled, don't write `useMemo`/`useCallback`/`React.memo`; flag existing ones for removal. Only memoize manually on profiler evidence.
3. **Effects**: for every `useEffect`, ask "is this synchronizing with an external system?" If it derives state → compute in render; resets on prop change → `key`; reacts to a user action → event handler; has stale closures → `useEffectEvent`.
4. **Component granularity**: extract only for reuse or real complexity; inline single-use logic-free wrappers.
5. **Data fetching**: server state goes through TanStack Query/SWR or Server Components, never hand-rolled `useState`+`useEffect`+`fetch`.
6. **Data shape**: stable data-derived keys, immutable updates, no disabled lint rules for these.
7. **State location**: shareable/bookmarkable state → URL params; ephemeral → local state.
8. **Seam**: domain logic in plain TS modules with no `react` import; components consume it.

## Related skills

- `next-expo-project-structure` — where the extracted domain logic, hooks, and feature code should live on disk.
