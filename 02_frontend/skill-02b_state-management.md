# SKILL 02-B — How to Manage State: Local, Context, and External Store
Category: Frontend and UI
Applies to: React, Next.js, React Native; Zustand, Jotai, Redux Toolkit

## What this skill covers
State management is the source of more React bugs, performance regressions, and architectural confusion than any other single topic. The mistake is treating every piece of state the same regardless of its scope, lifetime, or update frequency. This skill defines a clear decision tree for choosing the right state layer — local state, Context, or external store — and the correct implementation pattern for each.

## When to activate this skill
- When adding any new piece of shared state to a React application
- When `useState` is being prop-drilled more than two levels deep
- When evaluating whether to use Zustand, Redux, or Context
- When diagnosing unnecessary re-renders caused by state placement

## Core principles
1. **Use the minimum scope required.** If state is used in one component, use `useState`. Do not reach for Zustand for local toggle state.
2. **Server state is not client state.** Data that comes from the server belongs in React Query / SWR / tRPC, not in Zustand or Redux.
3. **Context is for low-frequency global values, not high-frequency updates.** Theme, auth user, locale — yes. Cart items that update on every keystroke — no.
4. **External stores are for genuinely global, non-server client state.** UI state shared across many components that changes frequently.
5. **Co-locate state with its consumers.** Push state down, not up, unless multiple components genuinely need the same state.

## Step-by-step guide
1. Ask: Is this state derived from server data? → Use React Query, tRPC, or SWR. Stop here.
2. Ask: Is this state used by only one component? → Use `useState` or `useReducer`. Stop here.
3. Ask: Is this state used by a small subtree of components (2-5)? → Lift state to their nearest common parent and pass via props. Stop here.
4. Ask: Is this state low-frequency (auth user, theme, locale) and global? → Use React Context. Stop here.
5. Ask: Is this state high-frequency, shared across many disconnected components, and not server state? → Use Zustand or Jotai. Stop here.
6. Implementing Context: create a dedicated `[Name]Context.tsx` file with Provider, typed value, and a custom hook (`useAuth()`, not `useContext(AuthContext)`).
7. Implementing Zustand: one store per domain (`useAuthStore`, `useCartStore`). Keep actions inside the store, not spread across components.
8. Never put server-fetched data into Zustand. Use React Query for caching, refetching, and synchronization.
9. Audit for unnecessary re-renders: use `React.memo`, `useMemo`, and `useCallback` only where profiling shows a problem — not as a default.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Store API response in Zustand and manually invalidate | Use React Query; it handles caching, invalidation, and refetching |
| Pass `isModalOpen` through 5 component layers | Use Zustand for shared UI state or lift state to the nearest common parent |
| Put everything in a global Context | Context re-renders all consumers on every update; use it only for low-frequency values |
| `useContext` called directly in every consumer | Create a custom hook (`useTheme()`) that wraps `useContext` with a null guard |
| One massive global Zustand store with all app state | One store per domain — `useAuthStore`, `useNotificationStore` |
| Derive values inside the store | Derive values with selectors or `useMemo` in components, keep store minimal |

## Stack-specific notes
**Next.js App Router:** Server Components have no state. Client Components (`"use client"`) use state. Do not reach for client state for data that can be fetched in a Server Component.
**React Native:** Same state rules apply. AsyncStorage is for persistence, not state management — integrate it explicitly in the store's persist middleware (Zustand supports this).
**Vite + React SPA:** Without Server Components, all state management is client-side. React Query for server state is even more important here since there is no server component layer.
**Go / Python / Node.js (server side):** Not applicable.

## Common mistakes
1. **Global state for local concerns.** A modal's open/close state that only one page uses does not need Zustand.
2. **Context for high-frequency updates.** A search input or scroll position in Context will re-render the entire tree on every keystroke.
3. **Mixing server state and client state.** A Zustand store that syncs with API responses requires manual synchronization that React Query does automatically.
4. **Using Redux in a new project with no existing Redux.** Redux Toolkit is powerful but heavy. Zustand and React Query cover 95% of cases with less boilerplate.
5. **Not creating a custom hook for Context.** Raw `useContext` calls scatter the context type across consumers and have no null guard.

## Checklist
- [ ] Server data managed with React Query / tRPC / SWR, not in Zustand
- [ ] Single-component state uses `useState` or `useReducer`
- [ ] State used by a small subtree is lifted to common parent with props
- [ ] Context used only for low-frequency global values (auth, theme, locale)
- [ ] Zustand used for high-frequency multi-component client UI state
- [ ] One Zustand store per domain, not one global store
- [ ] Context wrapped in a custom hook with null guard
- [ ] No prop drilling deeper than 2 levels without lifting or store
- [ ] No derived values stored in state — compute them with selectors or `useMemo`
- [ ] React DevTools or Zustand devtools used to verify update frequency
