---
name: react-frontend
description: Apply when writing or reviewing React (or React-based meta-framework) code — components, hooks, routing, state, forms, styling, accessibility. Triggers on .tsx/.jsx files, on imports of react/next/remix/vite/@tanstack, and on terms like "component", "hook", "route", "form", "Tailwind".
---

# React frontend

Apply these rules to any React or React-based code (Vite, Next.js, Remix, etc.).

## Non-negotiables

- **TypeScript, strict mode.** `any` is a code smell; `unknown` + a type guard is the answer.
- **Server state and client state are different.** Server state lives in React Query / SWR / RSC. `useState` is for ephemeral UI state. Don't put fetched data in `useState`.
- **Derive, don't sync.** If a value can be computed from props/state, compute it during render. Don't mirror it in another `useState` and keep them in sync with `useEffect`.
- **Effects are for synchronization with external systems** (DOM APIs, subscriptions, non-React libraries). Not for "run this after the user clicks." That's an event handler.
- **Accessibility is not optional.** Semantic HTML first, ARIA second. Every interactive element is keyboard-reachable and labelled.

## Defaults

- **Framework:** Next.js (App Router) for anything with routing/auth/SSR needs. Vite + React Router for SPAs.
- **Styling:** Tailwind for utility-first work. CSS Modules when a component genuinely needs scoped styles. No styled-components / Emotion for new code.
- **Components:** shadcn/ui as a baseline. Radix primitives directly when you need a behavior shadcn doesn't expose.
- **State:**
  - Server: TanStack Query (React Query).
  - Global client: Zustand. Redux only if the app already uses it.
  - Form: React Hook Form + zod.
  - Local: `useState` / `useReducer`.
- **Routing:** the framework's built-in router. No third-party routing inside Next.
- **Testing:** Vitest + React Testing Library for unit/component. Playwright for E2E. No Enzyme.
- **Icons:** lucide-react.
- **Dates:** date-fns or Temporal (when widely supported). No Moment.

## Anti-patterns — refuse without discussion

- `useEffect` to "fetch on mount" when React Query / RSC could do it.
- `useEffect` to sync two pieces of state. Lift, derive, or use a reducer.
- Destructuring `const { user } = useUserStore()` when only one field is needed and the store re-renders on any change. Use a selector.
- `key={index}` on a list of mutable items.
- Indexing styles by component name in global CSS.
- Spreading unknown props onto a DOM element (`<div {...props}>` where `props` came from a parent's API).
- Inline functions/objects passed as props to memoized children. Defeats the memoization. Either don't memoize or use `useCallback`/`useMemo`.
- Conditional hook calls. (The linter catches this; never silence it.)

## Quality bar

- Components are <150 lines. If a component grows past that, it's doing more than one thing — extract.
- Every form has validation messages that announce to screen readers.
- Every async UI has loading and error states. "Spinner forever" is a bug.
- No unhandled promise rejections in the console during a happy-path click-through.
- Lighthouse a11y score ≥ 95 on touched pages; CI fails if it regresses.
- Bundle size watched in CI; a single new dep that adds >50KB gzipped needs justification.

See [react-frontend/RULES.md](../../react-frontend/RULES.md) for the long-form rationale.
