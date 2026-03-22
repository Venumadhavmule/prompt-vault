# SKILL 02-E — How to Build Accessible (a11y) UI Components
Category: Frontend and UI
Applies to: React, Next.js, Vue, any web UI

## What this skill covers
Accessibility is not an add-on or a compliance checkbox — it is the test of whether a UI component actually works. Screen reader users, keyboard-only users, and users with motor or cognitive disabilities are 15-20% of users. An inaccessible component is a broken component. This skill covers the concrete, repeatable patterns for building keyboard-navigable, screen-reader-compatible, WCAG 2.1 AA-compliant UI components — without requiring specialist a11y expertise for every component.

## When to activate this skill
- When building any interactive UI component (button, form, modal, dropdown, tab)
- When building custom components that replace native HTML elements
- When using a headless UI library (Radix, Headless UI, Ark UI)
- When a PR review mentions accessibility issues
- During a security or quality audit

## Core principles
1. **Use semantic HTML first.** A `<button>` is better than a `<div role="button">` in every way. Let HTML carry the semantics.
2. **All interactive elements must be keyboard-operable.** Tab to focus, Enter/Space to activate, Arrow keys for composite widgets, Escape to close.
3. **Every non-text element needs a text alternative.** Icons, images, and icon-only buttons must have `aria-label` or `aria-labelledby`.
4. **Focus management is your responsibility.** When you open a modal, move focus into it. When you close it, return focus to the trigger.
5. **Color cannot be the only differentiator.** Error states, required fields, and status indicators must use more than color alone.

## Step-by-step guide
1. Prefer native HTML elements over custom ones: use `<button>`, `<a>`, `<input>`, `<select>`, `<details>` before reaching for div-based alternatives.
2. Ensure all interactive elements are focusable: buttons and inputs are focusable by default; custom elements need `tabIndex={0}`.
3. Implement keyboard handlers for custom interactive elements:
   - `onKeyDown` with `event.key === 'Enter'` or `' '` (Space) for button-like elements
   - Arrow keys for list/menu navigation
   - `Escape` to close dropdowns and modals
4. Apply ARIA attributes only when native semantics are insufficient:
   - `role="dialog"` and `aria-modal="true"` on modals
   - `aria-expanded` on accordion triggers
   - `aria-label` or `aria-labelledby` on icon-only buttons
   - `aria-live="polite"` for dynamic status messages
5. For modals: trap focus inside using `focus-trap-react` or the `inert` attribute on background content.
6. For forms: every input has a visible `<label>`. Use `htmlFor` on label pointing to input `id`. Errors associated with `aria-describedby`.
7. Ensure color contrast ratios meet WCAG AA: 4.5:1 for normal text, 3:1 for large text and UI components.
8. Test with keyboard-only navigation: tab through everything, activate every control.
9. Test with a screen reader: VoiceOver (macOS/iOS), NVDA/JAWS (Windows), or TalkBack (Android).

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `<div onClick={submit}>Submit</div>` | `<button type="submit">Submit</button>` |
| `<button><Icon /></button>` with no label | `<button aria-label="Close dialog"><Icon /></button>` |
| Open a modal, focus stays on background | Move focus to first focusable element inside the modal on open |
| `<input id="email">` with no associated label | `<label htmlFor="email">Email</label><input id="email" />` |
| Show error only with red border color | Show error with red border + error icon + text message |
| Error message in a `<div>` not connected to input | `<input aria-describedby="email-error" /><span id="email-error">...</span>` |

## Stack-specific notes
**Node.js / TypeScript (React):** Use Radix UI primitives — they handle most keyboard and ARIA patterns correctly out of the box. Build on top of them rather than from scratch.
**Vue:** Radix Vue or Headless UI for Vue provide similar accessibility primitives.
**Go / Python (server-rendered HTML):** Use semantic HTML and ensure form error states are associated with inputs via `aria-describedby` on every server-rendered form.
**All stacks:** axe-core (browser extension or integrated via jest-axe / vitest) is the standard automated a11y test tool. Add it to CI.

## Common mistakes
1. **ARIA without keyboard support.** Adding `role="button"` without keyboard activation handlers does not make the element accessible.
2. **Placeholder as the only label.** Placeholder text disappears on focus. Always have a visible or screen-reader-accessible label.
3. **Focus outline removal.** `outline: none` in CSS removes the only visible focus indicator for keyboard users. Use `:focus-visible` to remove outlines only for pointer users.
4. **Not returning focus after modal close.** Focus is lost, leaving keyboard users stranded.
5. **Testing only with automated tools.** axe-core catches ~40% of issues. Manual keyboard and screen reader testing catches the rest.

## Checklist
- [ ] Native HTML elements used wherever possible before ARIA overrides
- [ ] Every interactive element focusable via keyboard (Tab key)
- [ ] Keyboard handlers for Enter/Space on custom interactive elements
- [ ] Escape closes all popovers, dropdowns, and modals
- [ ] Every icon-only button has `aria-label`
- [ ] Every form input has an associated visible or accessible label
- [ ] Error messages connected to inputs via `aria-describedby`
- [ ] Modals trap focus and return focus on close
- [ ] Color contrast ≥ 4.5:1 for normal text verified
- [ ] Error/status states communicate with more than color alone
- [ ] Automated axe-core testing added to test suite
- [ ] Manual keyboard navigation test completed
