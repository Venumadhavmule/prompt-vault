# SKILL 02-I — How to Implement Internationalisation (i18n) Correctly
Category: Frontend and UI
Applies to: Next.js, React; next-intl, react-i18next, Paraglide

## What this skill covers
i18n is one of the easiest things to bolt on incorrectly and one of the most expensive to retrofit after the fact. The mistakes are consistent: hardcoded strings throughout the codebase, plural forms handled with if/else instead of ICU message syntax, date/number formats ignored entirely, and URL routing missing locale segments. This skill covers how to set up i18n correctly from the first commit so it scales to any number of locales without structural debt.

## When to activate this skill
- When the product needs to support more than one language
- Before writing any user-facing string in a new project (even if only one language initially)
- When adding i18n to an existing project with hardcoded strings
- When implementing locale-aware routing

## Core principles
1. **No hardcoded strings anywhere the user sees.** Every user-visible string is a key in a translation file from day one.
2. **ICU message format for plurals and variables.** `{count, plural, one {# item} other {# items}}` — not `count === 1 ? 'item' : 'items'`.
3. **Locale in the URL.** `/en/dashboard`, `/fr/dashboard` — locale must be part of the URL so it is shareable, indexable, and server-renderable.
4. **Numbers, dates, and currencies are locale-aware.** Use `Intl.NumberFormat`, `Intl.DateTimeFormat`, `Intl.RelativeTimeFormat` — never format these manually.
5. **Translation files are external to the component.** Messages live in JSON files, not in JSX strings.

## Step-by-step guide
1. Choose the i18n library: `next-intl` for Next.js App Router (recommended), `react-i18next` for Vite+React, Paraglide for type-safe compile-time messages.
2. Set up locale routing: in Next.js, use the `[locale]` dynamic segment under `app/[locale]/`. Middleware redirects root to the detected locale.
3. Create `messages/en.json` with a hierarchical structure: `{ "Auth": { "login": "Sign in", "logout": "Sign out" } }`.
4. Wrap the layout with the i18n provider and pass messages for the detected locale.
5. In components, use `t('Auth.login')` — never inline strings.
6. For interpolation: `t('welcome', { name: user.name })` with message `"welcome": "Welcome, {name}"`.
7. For plurals: `t('cartItems', { count: 5 })` with message `"cartItems": "{count, plural, one {1 item} other {# items}}"`.
8. For dates: use `new Date().toLocaleDateString(locale, { dateStyle: 'long' })` or the i18n library's date formatter.
9. Add TypeScript typing for translation keys: `next-intl` and Paraglide generate types automatically — enable this.
10. For SEO: add `<html lang={locale}>` and locale-specific `<meta>` tags and `hreflang` link tags.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `<h1>Welcome to the app</h1>` | `<h1>{t('home.welcome')}</h1>` |
| `count === 1 ? 'item' : 'items'` | ICU plural syntax in the message file |
| `new Date().toLocaleDateString()` without locale | `new Date().toLocaleDateString(locale, { dateStyle: 'medium' })` |
| All translations in one flat JSON object | Hierarchical JSON grouped by feature: `Auth`, `Dashboard`, `Billing` |
| No locale in URL: `/dashboard` for all locales | `/en/dashboard`, `/fr/dashboard` |
| Wait until multiple languages needed to add i18n | Set up i18n structure from the first user-facing string |

## Stack-specific notes
**Next.js App Router:** Use `next-intl`. Set up `i18n/routing.ts` with locale config, `middleware.ts` for locale detection, and wrap `app/[locale]/layout.tsx` with `NextIntlClientProvider`.
**Vite + React:** Use `react-i18next` with `i18next-http-backend` for lazy-loaded translation files.
**Vue / Nuxt:** Use `@nuxtjs/i18n` for Nuxt or `vue-i18n` for Vue. The principles (locale routing, no hardcoded strings, ICU plurals) are identical.
**Go, Python (server-rendered):** Use `gettext` / `.po` files for Python; Go's `golang.org/x/text` package. Return the locale from the request (Accept-Language header or cookie) and use it for all message lookups.

## Common mistakes
1. **Hardcoding strings in components from the start.** Retrofitting i18n into a codebase with 500 hardcoded strings is days of work. Add the infrastructure before writing any strings.
2. **Concatenating translated strings.** `t('hello') + ' ' + user.name` — different languages have different word orders. Use interpolation: `t('helloUser', { name })`.
3. **Missing plural support.** English has one plural rule. Arabic has six. Polish has four. Manual `count === 1` logic fails silently in every non-English locale.
4. **Missing date/number localisation.** Dates displayed as MM/DD/YYYY in markets that use DD/MM/YYYY cause direct user errors.
5. **Not adding `lang` attribute to `<html>`.** Screen readers and search engines use this attribute to determine language.

## Checklist
- [ ] i18n library installed and configured before any user-facing strings written
- [ ] Locale-based routing set up (`/[locale]/...`)
- [ ] Translation keys use hierarchical JSON grouped by feature
- [ ] No hardcoded user-visible strings in any component
- [ ] ICU message format used for all plurals and variable interpolation
- [ ] Dates formatted with `Intl.DateTimeFormat` and locale
- [ ] Numbers and currencies formatted with `Intl.NumberFormat` and locale
- [ ] `<html lang={locale}>` set in layout
- [ ] TypeScript types generated for translation keys
- [ ] Default locale configured and fallback to default when key missing
