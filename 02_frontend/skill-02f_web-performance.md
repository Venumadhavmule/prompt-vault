# SKILL 02-F — How to Optimise Web Performance and Core Web Vitals
Category: Frontend and UI
Applies to: Next.js, React, Vue, any web application

## What this skill covers
Performance is a feature, not an optimization. Core Web Vitals (LCP, CLS, INP) are Google's ranking signals and are direct measures of user experience. A 1-second delay in load time reduces conversions by 7%; a 100ms improvement increases revenue. This skill covers how to measure, diagnose, and improve performance across bundle size, rendering, images, fonts, and interaction responsiveness — in that order, because measuring first is non-negotiable.

## When to activate this skill
- When Lighthouse score is below 90 on any Core Web Vital
- When users report slow page loads
- When a new heavy dependency is being introduced
- During performance-focused code review
- Before a major product launch

## Core principles
1. **Measure before optimising.** Never guess what is slow. Use Lighthouse, WebPageTest, or Chrome DevTools Performance panel.
2. **LCP is the most important.** Largest Contentful Paint — the main content load time — must be under 2.5s. Everything else follows from this.
3. **Avoid layout shifts.** CLS (Cumulative Layout Shift) must be < 0.1. Every image and media element must have known dimensions before it loads.
4. **Bundle size is capped.** Initial JS bundle should not exceed 200KB compressed. Every dependency added must be justified against this budget.
5. **Defer what is not needed for the first paint.** Lazy-load below-the-fold content, below-the-fold components, and non-critical scripts.

## Step-by-step guide
1. Run Lighthouse in Chrome DevTools on the production URL (not localhost) and record the baseline scores for LCP, CLS, INP, and bundle size.
2. Diagnose LCP: is the LCP element an image? A hero heading? Check if it is render-blocked by JS or CSS. Ensure it is preloaded: `<link rel="preload" as="image">`.
3. Diagnose CLS: add `width` and `height` attributes to every `<img>` tag or use CSS `aspect-ratio`. Set explicit height on font containers before webfonts load.
4. Diagnose INP (Interaction to Next Paint): long tasks over 50ms block the main thread. Move heavy computation to `requestIdleCallback` or a Web Worker.
5. Audit the JS bundle: use `next/bundle-analyzer` or `vite-bundle-analyzer`. Identify and remove or lazy-load dependencies over 20KB.
6. Lazy-load below-the-fold components: use `dynamic(() => import('./Component'), { ssr: false })` in Next.js or `React.lazy()` + Suspense.
7. Optimise images: use `next/image` or equivalent. Serve WebP/AVIF formats. Size images at 2x their displayed dimensions for retina, not larger.
8. Optimise fonts: use `next/font` or `@font-face` with `font-display: swap`. Subset fonts to used character sets.
9. Preconnect to critical third-party origins: `<link rel="preconnect" href="https://cdn.example.com">`.
10. Deploy with HTTP/2 or HTTP/3. Enable CDN for static assets. Cache immutable assets with `Cache-Control: max-age=31536000, immutable`.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| Import `moment.js` (67KB) for date formatting | Use `date-fns` (tree-shakeable) or native `Intl.DateTimeFormat` |
| `<img src="hero.jpg">` with no dimensions | `<img src="hero.jpg" width="1200" height="600" alt="Hero">` — prevents CLS |
| Load all route JS upfront, even for unvisited routes | Code-split by route with dynamic imports |
| Block render with `<script src="analytics.js">` in `<head>` | Use `<script defer>` or load after first interaction |
| Use a 4000x3000px original image for a 400x300px thumbnail | Generate and serve correctly-sized thumbnails |
| Assume performance is good because it works locally | Always test on throttled simulated network (Lighthouse) |

## Stack-specific notes
**Next.js:** Use `next/image` (automatic WebP, lazy loading, size hints), `next/font` (eliminates layout shift from webfonts), and `next/dynamic` for lazy components. Analyze bundle with `ANALYZE=true pnpm build`.
**Vite + React:** Use `rollup-plugin-visualizer` for bundle analysis. Dynamic `import()` for code splitting. `vite-imagetools` for image transformation.
**Vue / Nuxt:** Nuxt has `useLazyAsyncData` for below-the-fold data loading. Vue's `defineAsyncComponent` for lazy components.
**Go / Python:** Server-side performance (TTFB) is a separate skill. This skill focuses on client-side Web Vitals.

## Common mistakes
1. **Optimising before measuring.** Spending time on micro-optimisations when the real bottleneck is an unoptimised 2MB hero image.
2. **Not setting image dimensions.** This is the single biggest source of CLS. It takes 5 seconds to fix and eliminates layout shift.
3. **Importing entire libraries for one function.** `import _ from 'lodash'` loads 70KB. `import debounce from 'lodash/debounce'` loads 2KB.
4. **Loading web fonts without `font-display: swap`.** Causes invisible text (FOIT) during font load — a direct CLS and UX failure.

## Checklist
- [ ] Lighthouse baseline recorded before optimization work
- [ ] LCP under 2.5s (measured on throttled connection, not localhost)
- [ ] CLS under 0.1 (all images have explicit width/height or aspect-ratio)
- [ ] INP under 200ms (no main thread tasks over 200ms)
- [ ] Initial JS bundle under 200KB compressed
- [ ] `next/image` or equivalent used for all images
- [ ] Below-the-fold components lazy-loaded with dynamic imports
- [ ] No `moment.js` — tree-shakeable alternatives used
- [ ] Fonts loaded with `font-display: swap` and subsetted
- [ ] CDN configured for static assets with immutable cache headers
- [ ] Bundle analyzer run to verify no unexpected large dependencies
