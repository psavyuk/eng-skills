# React frontend — rules

Long-form companion to [skills/react-frontend/SKILL.md](../skills/react-frontend/SKILL.md).

## Component design

A component does one of three things:

1. **Renders a layout / page** — composes other components, owns route-level data fetching, owns page-level state.
2. **Renders a piece of UI** — pure-ish, props in, JSX out. No data fetching.
3. **Wraps behavior** — a custom hook or a render-prop component. No visual output.

Mixing the three is the most common reason React code becomes unmaintainable.

## State, ranked from best to worst

1. **Server state in a query library** (React Query, RSC cache). Cached, deduped, automatically refetched.
2. **URL state** for anything the user might want to share, bookmark, or refresh. Filters, tabs, pagination.
3. **Local component state** with `useState` / `useReducer`. Ephemeral, never persisted.
4. **Global client store** (Zustand) for state that crosses many components and isn't from the server. Use sparingly — most "global" state turns out to be server state.
5. **Context** for dependency injection (theme, auth user object, feature flags). Not for high-frequency updates.

If you find yourself reaching for `useEffect` to keep two of the above in sync, pick a single source and derive the rest.

## Effects

Three legitimate uses for `useEffect`:

1. Subscribing to an external system (DOM event, window resize, WebSocket).
2. Imperatively calling a non-React library (chart, map, video player).
3. Cleaning up #1 or #2.

Everything else has a better home: event handlers for user actions, RSC / query libraries for data, render-time computation for derived values.

If an effect has an empty dependency array and runs on mount, ask: should this be in a layout/loader instead?

## Data fetching

- **Server components (Next App Router):** fetch in the RSC, pass data down. Default choice.
- **Client components:** React Query. Never raw `useEffect` + `fetch`.
- **Mutations:** `useMutation` with optimistic updates only when the operation is fast and reversible.
- **Cache keys:** always arrays, always specific. `['user', userId, 'orders']` not `['orders']`.

## Forms

React Hook Form + zod. Pattern:

```tsx
const schema = z.object({...});
const form = useForm({ resolver: zodResolver(schema) });
```

- Validation messages live next to the field, announced via `aria-describedby`.
- Submit button shows pending state from `form.formState.isSubmitting`.
- On server error, set field errors with `form.setError`, not toasts.

## Styling

Tailwind first. Convention:

- Use `cn()` (clsx + tailwind-merge) for conditional classes.
- Extract a component before extracting a class name. A repeated `flex items-center gap-2 ...` chain is fine; the third time, make it a component.
- Theme via CSS variables under a `:root` / `.dark` selector, not via Tailwind's dark variants for every color.

No CSS-in-JS for new code. Runtime cost, SSR pain, no clear advantage.

## Accessibility

The 80/20:

- Use the semantic element first (`<button>`, `<a>`, `<label>`, `<nav>`). ARIA roles fix wrong elements; semantic elements never needed fixing.
- Every input has a label. `aria-label` is a fallback, not a default.
- Focus management: opening a modal moves focus into it; closing returns focus to the trigger.
- Keyboard nav: tab through every interactive element in a page and confirm you can complete the main flow without a mouse.
- Color is never the only signal. Error states have an icon or text, not just red.

## Performance

In order:

1. Don't fetch what you don't render.
2. Don't render what's not visible (virtualize long lists).
3. Don't re-render what hasn't changed (selectors, `React.memo`, stable refs).
4. Code-split routes and heavy components (`dynamic`/`lazy`).
5. Profile before optimizing. React Profiler is free.

Premature `useMemo`/`useCallback` is the second-worst kind of "optimization" (after `React.memo` on everything). They have a cost too.

## Testing

- **Unit / component:** Vitest + React Testing Library. Query by role and label, not by class or test-id (test-id is a fallback when the DOM is unhelpful).
- **Integration:** msw to mock the network in component tests. Real fetch, fake responses.
- **E2E:** Playwright for critical user journeys (sign in, checkout, primary CRUD). One per journey, not per page.

## Folder layout (small/medium apps)

```
src/
  app/                # Next App Router or routes
  components/         # Cross-feature, presentational
  features/<name>/    # Feature-scoped: components, hooks, api, types
  lib/                # Cross-cutting helpers (cn, fetchers, date utils)
  styles/
```

`features/` is the unit of "this code belongs together." Don't pre-split components/hooks/types into top-level folders that grow forever.

## What this skill is *not*

- React Native (different platform constraints).
- Build tooling beyond defaults (Webpack/Rollup configuration belongs in a separate doc).
- Server-side concerns of meta-frameworks (use backend-engineer for API design).
