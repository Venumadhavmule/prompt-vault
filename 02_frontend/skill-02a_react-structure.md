# SKILL 02-A — How to Structure a React Application Correctly
Category: Frontend and UI
Applies to: React, Next.js, Remix, Vite + React, any React-based project

## What this skill covers
React has no enforced structure, which means most React codebases grow into an unnavigable tangle of co-located components, scattered state, and layerless business logic. This skill defines the correct directory structure and file organization for any React application from the first commit — structured for scale regardless of project size at the start, because restructuring at scale is 10x more expensive than starting right.

## When to activate this skill
- When creating a new React project or feature folder
- When a codebase lacks an obvious structure and you need to orient yourself
- When components have grown beyond simple UI and contain business logic
- When the same data-fetching logic appears in more than one component

## Core principles
1. **Separate UI from logic.** Components render. Services fetch. Hooks encapsulate stateful behavior. Never mix concerns inside a component.
2. **Colocation over centralization for small units.** A hook used only by one component lives next to that component.
3. **Centralization for shared code.** Shared hooks, utilities, and types live in clearly named shared directories, not scattered across feature folders.
4. **Feature folders over layer folders at scale.** At small scale, group by layer (`components/`, `hooks/`). At medium-large scale, group by feature (`features/auth/`, `features/billing/`).
5. **Components should not know where data comes from.** Pass data down via props or context — do not fetch inside leaf components.

## Step-by-step guide
1. Establish the top-level structure (use feature-first once you have 3+ distinct features):
   ```
   src/
     app/           # Next.js App Router pages, or React Router routes
     components/    # Shared UI components used by 2+ features
     features/      # One folder per major feature
       auth/
         components/   # UI components for this feature only
         hooks/        # Hooks for this feature only
         services/     # API calls for this feature
         types.ts      # Types for this feature
     hooks/         # Global shared hooks
     lib/           # Third-party clients (db, stripe, analytics)
     types/         # Global TypeScript types
     utils/         # Pure utility functions
   ```
2. Each component file contains: one exported component, its local types, and nothing else.
3. Co-locate component-specific hooks and sub-components in the same feature folder.
4. Put all API calls in service files (`auth/services/authService.ts`), never inside components.
5. Put all shared UI primitives in `components/ui/` (Button, Input, Modal, etc).
6. Never import from a child feature into a parent feature — features should be independent.
7. Use barrel `index.ts` files only for public APIs of feature folders, not to re-export everything.
8. Every component that fetches data should have a corresponding loading and error state.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Build a `components/` folder with 80 files flat | Organize into `components/ui/`, `components/layout/`, `features/[name]/components/` |
| Fetch data inside a UI component | Fetch in a service or React Query hook, pass result as props |
| Put business logic inline in event handlers | Extract to a service function or a custom hook |
| Import a feature-specific hook into a different feature | Move the hook to `hooks/` (shared) if multiple features need it |
| One gigantic `App.tsx` with all routes | Each route is its own file in `app/` or `pages/` with its own data loading |
| Types scattered across component files | Feature-level `types.ts` and global `types/` directory |

## Stack-specific notes
**Next.js App Router:** Use the `app/` directory for routes. Use `_components/` prefix for route-local components not exported. `page.tsx` should be thin — import from feature modules.
**Vite + React:** You are responsible for your own router. React Router v6 with nested routes mirrors the App Router mental model. Apply the same feature-folder structure.
**Remix:** Remix colocates data loading with routes; still extract reusable logic to `lib/` or `services/`.
**Go:** Not applicable.
**Python:** Not applicable.

## Common mistakes
1. **Everything in `components/`.** When all 200 components are in one flat directory, every search requires reading a hundred file names.
2. **Fetching in leaf components.** A `<UserAvatar>` should not call `fetch('/api/users/me')`. It should receive `user` as a prop.
3. **Giant `utils.ts` files.** A catch-all utils file is a sign of deferred organization. Split into specific utilities by domain.
4. **Mixing page logic with component logic.** A page-level component handles routing and data loading; a feature component handles UI. Keep them separate.

## Checklist
- [ ] Feature-folder structure established for projects with 3+ features
- [ ] No business logic or data fetching inside leaf UI components
- [ ] Shared UI components in `components/ui/`
- [ ] All API calls in service files, not in components
- [ ] Feature-level `types.ts` for feature-specific types
- [ ] Global `types/` and `hooks/` directories for shared code
- [ ] Barrel `index.ts` files used only for intentional public APIs
- [ ] No cross-feature imports (feature A does not import from feature B)
- [ ] Every data-fetching component has loading and error state
- [ ] Linter and TypeScript configured and passing
