# SKILL 02-C — How to Build with Next.js App Router Correctly
Category: Frontend and UI
Applies to: Next.js 13+, App Router architecture

## What this skill covers
Next.js App Router introduced a fundamentally different rendering model that most developers apply incorrectly — defaulting to Client Components when Server Components are the right choice, placing data fetching in the wrong layer, and misunderstanding the boundary between server and client. This skill defines exactly when to use Server vs Client Components, how to structure layouts, how to co-locate loading and error states, and how to handle data fetching at every layer correctly.

## When to activate this skill
- When creating any new Next.js feature with the App Router
- When deciding whether a component needs `"use client"`
- When deciding where to fetch data for a route
- When a Server Component accidentally includes browser-only code
- When debugging hydration errors

## Core principles
1. **Server Components are the default.** Only add `"use client"` when the component needs browser APIs, event listeners, or hooks.
2. **Data fetching belongs in Server Components.** Fetch in layout or page Server Components, pass data down to Client Components via props.
3. **Push the client boundary as far down as possible.** A page should be a Server Component; only its interactive leaf nodes should be Client Components.
4. **Colocate loading.tsx and error.tsx next to page.tsx.** Do not handle loading/error state manually when Next.js provides it via filesystem convention.
5. **Layouts share state across routes; pages are stateless on navigation.** Understand what re-renders on navigation and what does not.

## Step-by-step guide
1. Start every new component as a Server Component (no `"use client"` directive).
2. Add `"use client"` only when the component uses: `useState`, `useEffect`, `useContext`, `onClick` / `onChange` event handlers, or browser-only APIs (`window`, `localStorage`).
3. For data fetching, use `async` Server Component functions directly: `const data = await fetchData()` in `page.tsx` or `layout.tsx`.
4. Pass fetched data to Client Components via props — Client Components cannot call async functions on the server.
5. Use `loading.tsx` alongside every `page.tsx` that fetches data to provide automatic Suspense UI.
6. Use `error.tsx` alongside every `page.tsx` that might throw to provide automatic error handling.
7. For interactive data (mutations): use Server Actions (`"use server"` functions) for form submissions and mutations, or use client-side React Query for optimistic updates.
8. For layouts: `layout.tsx` wraps a segment and all its children — fetch shared data (auth session, user profile) here once.
9. Protect routes using middleware (`middleware.ts` at root) or by checking auth in the layout Server Component.
10. Route groups (`(group)/`) organize routes without affecting the URL structure — use them to share layouts across related routes.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Add `"use client"` to every component by default | Start as Server Component; add `"use client"` only when required |
| Fetch data in a `useEffect` inside a Client Component | Fetch in a Server Component or use React Query with a Server Action |
| Manually manage loading state with `useState` | Add `loading.tsx` file next to `page.tsx` for automatic Suspense |
| Manually catch errors and render error UI | Add `error.tsx` file next to `page.tsx` for automatic error boundaries |
| Import a Server Component into a Client Component | Pass Server Component output as `children` prop to the Client Component |
| Use `useRouter().push()` for every navigation | Use `<Link>` for navigations that should be pre-fetched |

## Stack-specific notes
**Next.js 13+ App Router:** The ISR (Incremental Static Regeneration) flag `revalidate` can be set per-`fetch` call or per-route segment. Use `{ next: { revalidate: 60 } }` for static with timed refresh.
**Node.js server:** App Router Server Components run in a Node.js environment — you can import server-only modules here. Use `server-only` package to enforce this.
**Python / Go:** Not applicable.

## Common mistakes
1. **Adding `"use client"` at the top of `app/layout.tsx`.** This turns the entire app into client-rendered, eliminating all Server Component benefits.
2. **Fetching in `useEffect` when a Server Component can fetch it.** Adds a waterfall, a loading state, and a client-side bundle increase for no benefit.
3. **Using cookies or headers in a Client Component.** These are server-only. Read them in a Server Component or Server Action.
4. **Not using `loading.tsx`.** Manually implemented loading states are error-prone and duplicate what Next.js provides for free.
5. **Putting the entire layout in a Client Component.** A shared nav that has one interactive dropdown should have the nav as Server Component and only the dropdown as Client Component.

## Checklist
- [ ] Default component type is Server Component; `"use client"` is added only when necessary
- [ ] Data fetching done in Server Components, not in `useEffect`
- [ ] `loading.tsx` exists alongside every data-fetching `page.tsx`
- [ ] `error.tsx` exists alongside every `page.tsx` that might throw
- [ ] Auth checked in middleware or root layout Server Component
- [ ] Client boundary pushed to the lowest possible component
- [ ] No browser-only APIs (`window`, `localStorage`) in Server Components
- [ ] Server Actions used for form mutations instead of API route + client fetch
- [ ] Route groups used to organize layouts without URL pollution
- [ ] `metadata` exported from `page.tsx` for SEO
