# SKILL 02-J — How to Build a Vue / Nuxt Application (Differences from React)
Category: Frontend and UI
Applies to: Vue 3, Nuxt 3, Composition API

## What this skill covers
Vue and React share the component model but differ significantly in reactivity, state, routing, and the server-rendering layer. Engineers who switch between the two fail by applying React patterns literally to Vue. This skill covers Vue 3's Composition API, Nuxt 3's file-system routing and universal rendering, and the critical differences from React that, if missed, produce subtle bugs and non-idiomatic code.

## When to activate this skill
- When building any Vue 3 or Nuxt 3 project
- When a React-experienced developer is working in a Vue codebase
- When deciding between composables, Pinia, and local reactive state
- When configuring server-side rendering in Nuxt

## Core principles
1. **Vue's reactivity is automatic.** `ref()` and `reactive()` create reactive state that auto-tracks and auto-updates — there are no dependency arrays like `useEffect`.
2. **Composables replace custom hooks.** A composable is a function beginning with `use` that contains reactive state and logic — the direct Vue equivalent of a React custom hook.
3. **Pinia replaces Zustand/Redux.** Pinia is the official Vue store. Use it exactly where you would use Zustand in React.
4. **`<template>` is the component's render output, not a function.** JSX is possible in Vue but non-idiomatic; use Single File Components (`.vue` files).
5. **`computed()` is not `useMemo()`.** Vue's `computed()` is automatically reactive — it re-runs when its dependencies change without any manual dependency list.

## Step-by-step guide
1. Use the Composition API exclusively in Vue 3 — not the Options API. `setup()` or `<script setup>` is the correct pattern.
2. Use `<script setup>` syntax — it eliminates boilerplate and enables direct variable exposure to the template:
   ```vue
   <script setup lang="ts">
   const count = ref(0);
   const doubled = computed(() => count.value * 2);
   </script>
   ```
3. Reactive state: use `ref()` for primitive values; `reactive()` for objects you want to destructure using `toRefs()`.
4. Side effects: use `watchEffect()` for effects that should re-run when reactive dependencies change. Use `watch()` to react to a specific source value.
5. Lifecycle hooks: `onMounted()`, `onUnmounted()`, `onBeforeMount()` — called inside `<script setup>` directly.
6. Extract shared stateful logic into composables in `composables/` directory: `useAuth.ts`, `useCart.ts`.
7. For global state: use Pinia. One store per domain. Actions are methods, state is reactive automatically.
8. **Nuxt 3 specifics:**
   - File-based routing: `pages/dashboard.vue` → `/dashboard`
   - `useFetch()` / `useAsyncData()` for server-side data fetching (not `useEffect`)
   - Auto-imports: components in `components/`, composables in `composables/`, utils in `utils/` are auto-imported
   - Universal rendering: pages render on server first, then hydrate on client

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `useEffect` equivalent — manually track dependencies | `watchEffect(() => { /* runs when reactive deps change */ })` — no dependency array |
| `useState` at component level for all state | `ref()` for primitives, `reactive()` for objects |
| Zustand for global state | Pinia — the official, idiomatic Vue store |
| `import UserCard from './UserCard.vue'` manually in every file | Nuxt auto-imports components from `components/` directory |
| `fetch()` in `onMounted` for server data | `const { data } = await useFetch('/api/users')` — server-rendered, cached, type-safe |
| Options API in new Vue 3 code | Composition API with `<script setup>` — always |

## Stack-specific notes
**Nuxt 3 (preferred for Vue SSR):** Use `useFetch` and `useAsyncData` — they work on both server and client and avoid hydration mismatches. Use `useState` (not Vue's `ref`) for state that must be shared across SSR and CSR.
**Vue 3 + Vite (SPA):** Identical to Nuxt minus the routing convention and auto-imports. Add `unplugin-auto-import` for composable auto-imports.
**Node.js, Python, Go:** Not applicable.

## Common mistakes
1. **`.value` forgotten.** `ref()` values are accessed as `.value` in `<script setup>` but template auto-unwraps. Forgetting `.value` in the script block silently passes the ref object, not the value.
2. **Destructuring `reactive()` objects directly.** `const { name } = reactive({ name: 'Alice' })` breaks reactivity. Use `toRefs()` when destructuring.
3. **Mutating Pinia state directly outside a store action.** Mutations in strict mode throw. Always use an action.
4. **Using `useEffect` patterns — manually setting state after fetch.** Vue's `useFetch` and `watchEffect` eliminate this pattern entirely.

## Checklist
- [ ] Composition API with `<script setup>` used — not Options API
- [ ] `ref()` for primitive values, `reactive()` with `toRefs()` for objects
- [ ] `computed()` used for derived values — no manual memo
- [ ] `watchEffect()` used for side effects — no manual dependency arrays
- [ ] Pinia stores created per domain for global state
- [ ] Shared logic extracted to composables in `composables/`
- [ ] Nuxt `useFetch` / `useAsyncData` used for data fetching (not `onMounted` + `fetch`)
- [ ] File-based routing used correctly with `pages/` directory
- [ ] No direct access to reactive `.value` inside `<template>` (auto-unwrapped)
- [ ] TypeScript enabled with `lang="ts"` on `<script setup>`
