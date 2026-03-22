# SKILL 02-H — How to Design a Component Library with shadcn/ui or Radix
Category: Frontend and UI
Applies to: React, Next.js; shadcn/ui, Radix UI, Headless UI

## What this skill covers
Building a component library on top of shadcn/ui or Radix primitives is the fastest path to a production-quality, accessible, themeable UI system. The mistake most teams make is either treating shadcn components as black boxes (fragile) or rebuilding them from scratch (expensive). This skill covers how to own the component layer correctly: consuming headless primitives, extending them for your design system, building consistent variant APIs, and maintaining the library as requirements evolve.

## When to activate this skill
- When bootstrapping a new project's UI component system
- When adding a new reusable component to an existing design system
- When standardising a component API across a codebase
- When a component needs accessibility that a bare div cannot provide

## Core principles
1. **Own the components.** shadcn/ui copies components into your codebase — you own the code and can modify it. This is a feature, not a limitation.
2. **Radix primitives handle all behavior; you handle visual.** Never add keyboard handling or ARIA to a component built on Radix — it is already there.
3. **Consistent variant API across all components.** Use class-variance-authority (CVA) so every component has the same `variant`, `size`, and `intent` prop pattern.
4. **Components accept a `className` prop.** Every component must allow the consumer to add classes for one-off overrides without forking the component.
5. **Never put business logic inside UI components.** A `<Button>` should not know about auth state, API calls, or routing.

## Step-by-step guide
1. Initialize shadcn/ui in the project: `npx shadcn@latest init` — choose your style (Default or New York), set globals.css, configure path aliases.
2. Add components as needed: `npx shadcn@latest add button dialog form` — each is copied into `src/components/ui/`.
3. Do not modify shadcn-generated components for one-off use cases. Add a custom wrapper component instead.
4. For project-specific variants, extend the component using CVA:
   ```ts
   const buttonVariants = cva("base classes", {
     variants: {
       intent: { primary: "bg-blue-600 text-white", danger: "bg-red-600 text-white" },
       size: { sm: "h-8 px-3 text-sm", md: "h-10 px-4", lg: "h-12 px-6 text-lg" },
     },
     defaultVariants: { intent: "primary", size: "md" },
   });
   ```
5. Every component accepts `className` as a prop and passes it to the root element via `cn(baseClasses, className)`.
6. When building compound components (Dialog with trigger, content, close), keep the compound pattern from Radix rather than flattening into a single component.
7. Document each component's props with TypeScript types — no `any` props, ever.
8. Build a Storybook or simple `components/showcase.tsx` page to visually verify every variant of every component.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Install shadcn, never touch the component files | Own the components — read and understand every component you use |
| Hardcode `bg-blue-600` inside the Button component | Use CVA variants that map to design tokens from `tailwind.config` |
| Build a custom accessible Dialog from scratch | Use Radix `Dialog` primitive — focus trap, ARIA, keyboard handling is built in |
| Pass a click handler prop into `<Button onClick={doAPICall}>` | Keep API call in parent; `<Button onClick={handleClick}>` passes to parent logic |
| Duplicate a component with minor change instead of adding a variant | Add a variant to the CVA definition |
| No `className` prop on custom components | Every component: `<Component className={cn(base, className)} />` |

## Stack-specific notes
**Next.js:** shadcn/ui components are Client Components by default (they use hooks). Wrap in a Server Component layout that passes data down if needed.
**Vite + React:** Identical setup. Shadcn works without Next.js.
**Vue:** shadcn-vue and Radix Vue provide equivalent primitives. The CVA pattern works identically.
**Go / Python:** Not applicable.

## Common mistakes
1. **Copying shadcn components and never reading them.** If you encounter an issue, you need to understand the component internals to fix it.
2. **Not using CVA for variant management.** Managing variants with manual conditional class strings creates inconsistency and merge conflicts.
3. **Re-implementing focus trapping in a dialog.** Radix Dialog already does this. Adding custom focus management on top creates double-trapping bugs.
4. **Putting routing logic (`useRouter`) inside a UI component.** UI components should be routing-agnostic — pass an `href` or `onClick` from outside.

## Checklist
- [ ] `shadcn@latest init` run with correct config (globals.css, path aliases)
- [ ] Components copied into `src/components/ui/` — owned and readable
- [ ] CVA used for all component variant definitions
- [ ] Every component accepts and applies a `className` prop
- [ ] No business logic (API calls, routing, auth) inside UI components
- [ ] Compound components (Dialog, Dropdown) use Radix's compound pattern
- [ ] All props typed with TypeScript — no `any`
- [ ] Visual showcase or Storybook exists to verify all variants
- [ ] Design tokens from `tailwind.config` used in variant classes
- [ ] Linter passes on all component files
