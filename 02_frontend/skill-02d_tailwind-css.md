# SKILL 02-D — How to Use Tailwind CSS Without Creating a Mess
Category: Frontend and UI
Applies to: Tailwind CSS v3+, any React/Vue/Svelte project using Tailwind

## What this skill covers
Tailwind enables fast styling but punishes undisciplined usage. Projects start clean and within six months have: class strings with 40 utilities per element, inconsistent spacing, duplicated patterns across dozens of components, and no relationship between the design system and the code. This skill covers how to use Tailwind properly — with consistent design tokens, abstracted component patterns, and conventions that keep long-term maintainability high.

## When to activate this skill
- When starting a new Tailwind-based project
- When adding a new UI component that others will reuse
- When a className string exceeds ~15 utilities
- When the same utility combination appears in multiple separate components

## Core principles
1. **Design tokens first.** Configure `tailwind.config` with the actual design system values (colors, spacing, font sizes) before writing components.
2. **Extract repeated combinations to components, not CSS.** When the same set of utilities appears three or more times, extract a React component, not a CSS `@apply`.
3. **Avoid `@apply` in global CSS.** It defeats Tailwind's purpose and breaks tree-shaking. The only acceptable use is for third-party HTML you cannot control.
4. **Variant classes must always be complete strings.** Dynamic class assembly with string interpolation breaks Tailwind's static analysis and purging.
5. **Use `cn()` for conditional classes.** The `clsx` + `tailwind-merge` pattern (`cn` utility) is the standard for conditional and merged class handling.

## Step-by-step guide
1. Configure `tailwind.config.ts` before writing any component:
   - Set brand colors under `colors`
   - Set typography scale under `fontSize`
   - Set spacing system under `spacing` if deviating from defaults
   - Set breakpoints under `screens` to match the design
2. Install and configure `clsx` and `tailwind-merge`. Create a `cn()` utility:
   ```ts
   import { clsx } from 'clsx'; import { twMerge } from 'tailwind-merge';
   export function cn(...inputs) { return twMerge(clsx(inputs)); }
   ```
3. Use `cn()` for all conditional class application — never string interpolation for toggling classes.
4. Write class strings in a consistent order: layout → sizing → spacing → typography → color → border → shadow → animation.
5. When a class string exceeds 12 utilities, extract to a named component.
6. For reusable variants (size, color, intent variants), use `class-variance-authority` (CVA) to define the variant map.
7. Never interpolate Tailwind class names dynamically: `text-${color}-500` will not be detected by Tailwind's scanner. Use a lookup map instead.
8. Run `prettier-plugin-tailwindcss` to auto-sort class names consistently.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `className={`text-${variant}`}` — dynamic interpolation | Use a lookup object: `const variantClass = { primary: 'text-blue-600', ... }[variant]` |
| 40 utility classes on one `<div>` | Extract a `Card` or `ButtonGroup` component |
| `@apply btn-primary` in CSS file | Extract a `<Button variant="primary">` React component |
| Hardcode `#3b82f6` as an arbitrary value `text-[#3b82f6]` | Define `colors.primary` in `tailwind.config` and use `text-primary` |
| `text-gray-500` in some places, `text-neutral-500` in others | Pick one gray scale in config and use it everywhere |
| No `cn()` utility, concatenate classes with template literals | Always use `cn()` to merge and deduplicate classes |

## Stack-specific notes
**Next.js:** Import globals (including Tailwind `@tailwind` directives) in `app/layout.tsx`. Use `next/font` for typography alongside Tailwind's font config.
**Vite + React:** Add the Tailwind PostCSS plugin. Identical usage otherwise.
**Vue / Nuxt:** Tailwind usage is identical. UnoCSS is a popular Tailwind-compatible alternative worth evaluating for Nuxt.
**Go / Python (server templates):** Tailwind can be used with server-rendered HTML. Use the CLI in watch mode during development. The same class string rules apply.

## Common mistakes
1. **Dynamic class interpolation.** `bg-${color}` does not work — Tailwind scans for complete class strings. This is the most common Tailwind bug.
2. **Mixing Tailwind with custom CSS classes for the same element.** Conflict resolution is confusing. Use one or the other; `tailwind-merge` handles Tailwind-vs-Tailwind conflicts.
3. **Ignoring `tailwind.config`.** Using arbitrary values (`text-[14px]`) when the design system has defined values means the design system is not being used.
4. **Not using prettier-plugin-tailwindcss.** Unsorted class strings make diffs unreadable and comparison impossible.

## Checklist
- [ ] `tailwind.config.ts` configured with design tokens (colors, fonts, spacing)
- [ ] `cn()` utility created using `clsx` + `tailwind-merge`
- [ ] `prettier-plugin-tailwindcss` installed and configured
- [ ] No dynamic class interpolation with template literals
- [ ] Repeated class combinations (3+ occurrences) extracted to components
- [ ] `class-variance-authority` used for components with multiple variants
- [ ] No `@apply` in application CSS
- [ ] Arbitrary values used only when design tokens do not cover the need
- [ ] Consistent class ordering (layout → size → space → type → color → border)
- [ ] Single color scale used consistently (no mixing `gray` and `neutral`)
