# Module 29 — Performance (Strategy, Budgets & the SPA Comparison)

> Phases 24/45/46/76 + case study 47. A serious performance program for Astro: budgets, Core Web Vitals, the post-load mental model — and a measured comparison of Heavy SPA vs Astro islands.

## The web performance mental model (what happens after a user opens a page)

```text
DNS/TCP/TLS → HTML (TTFB)        ← server/static + CDN
  → parse HTML → CSS (render-blocking) → fonts
  → paint text/hero (LCP)        ← images, server HTML completeness
  → JS downloads → hydration     ← island directives
  → interaction readiness (INP)  ← hydration CPU, long tasks
  → stability (CLS)              ← dimensions, late-loading UI
```

**NETWORK → HTML → CSS → JS → IMAGES → FONTS → HYDRATION → INTERACTION.** Every budget question maps to a stage.

## Core Web Vitals for Astro

| Metric | What drives it in Astro | Astro-specific levers |
|---|---|---|
| **LCP** | TTFB + hero render | static/CDN pages, `routeRules` cache (Module 34), optimized hero `<Image>` (no lazy), preload |
| **CLS** | late UI | image `width/height`, font fallbacks, island fallback slots (`client:only`, `server:defer` skeletons) |
| **INP** | hydration + handlers | directive strategy (Module 08), small islands, avoid main-thread work |

## The JavaScript budget (make it explicit)

Example budget (adjust per product, keep it written):

| Category | Budget |
|---|---|
| Content pages (blog/docs) | **0 KB** framework JS; total JS < 20 KB gzip |
| Marketing page | total JS < 50 KB gzip; ≤ 1 `client:load` island |
| App dashboard | framework shared chunk + islands ≤ 170 KB gzip total |
| Third-party | ≤ 20 KB; async; facades for embeds |

Enforcement: CI bundle report (Rolldown/`astro build` output analysis + size-limit tooling), quarterly audits.

## Images, fonts, CSS

- Images: Module 25 (formats, srcset, LCP hero rules).
- Fonts: subset + self-host, `font-display`, ≤ 2 families/4 weights; or Astro Fonts API.
- CSS: global CSS small (Module 06); critical path clean; container queries over JS resize.

## Caching & delivery (link to Module 34)

- Static: immutable `/_astro/*`, CDN-edge HTML.
- SSR: `routeRules` `maxAge`/`swr`/tags; `private, no-store` for personalized.
- Preconnect only where you actually fetch (fonts/CDN origins).

## Case study: Heavy SPA vs Astro islands (methodology + expected results)

**Marketing page spec:** hero + 3 features + pricing table + testimonial carousel + newsletter.

| | Implementation A: full React SPA | Implementation B: Astro + islands |
|---|---|---|
| HTML | empty root + boot JS | complete HTML at TTFB |
| JS (gzip, typical) | 150–300 KB (React + router + carousel lib) | ~0–15 KB (carousel `client:visible` + form script) |
| LCP | waits boot + render | server HTML + hero image |
| INP | after full hydration | isolated island hydration |
| SEO | needs SSR/prerender setup | native |
| A11y | depends on SPA components | semantic baseline |
| Carousel cost | in critical path | visible-only |

**Measure it yourself:** Lighthouse + WebPageTest on both; compare TTFB, LCP, CLS, INP, JS bytes. The lesson isn't the exact numbers — it's *where the numbers come from* (hydration scope).

**Hydration cost detail:** downloading 40 KB React is one cost; running `hydrate()` over a large tree on a mid-tier Android is the invisible one. Islands bound both.

## Bundle analysis workflow

```bash
npm run build
# inspect dist/_astro/*.js sizes
# use rollup-plugin-visualizer via vite config if needed
```

Check per route: which chunks? which islands? Any accidental framework import in a script?

## Common Mistakes

**BAD:** "It feels fast on my M3 Max."

**GOOD:** test mid-tier mobile + 4G; track field data (Module 31 Web Vitals).

**BAD:** five `client:load` islands in the header "for consistency."

**GOOD:** budget per page; directives per interaction.

**BAD:** 800 KB hero PNG in `public/`.

**GOOD:** `astro:assets`.

**BAD:** third-party chat + analytics + A/B scripts all sync in head.

**GOOD:** facades, `client:idle`/`visible`, consent-gated (Module 31).

## Security Notes

- Perf tricks with security edges: prefetching authenticated URLs can leak timing; `preload` hints reveal priorities — keep private responses `no-store`.
- Third-party scripts are an XSS/perf/privacy triple threat — audit them like code.

## Performance Notes

- This module *is* the performance note. Keep the budget visible in the repo (`PERFORMANCE.md`), review PRs against it.

## Exercises

**Beginner.** Measure Lighthouse on your blog; get LCP < 2.5s on simulated mobile.

**Intermediate.** Cut a page's JS by 50% via directive changes alone; document bytes before/after.

**Production.** Implement the SPA-vs-islands comparison on two real routes; produce a one-page report with charts for stakeholders.

**Architecture Challenge.** Product wants "app-like everything": persistent player, global cart, live chat, animated page transitions. Budget it.

> **Review:** persistent player: `transition:persist` island (Module 27); cart badge: `server:defer` + tiny island; chat: `client:idle`; transitions: ClientRouter site-wide ≈ +3 KB; total budget stays under 100 KB on content pages. "App-like" is a schedule of small costs, not one big one.

**Debugging Challenge.** INP is fine on desktop, awful on mobile; LCP fine. A `client:visible` comments widget hydrates a 2,000-comment list synchronously as users scroll. Fix: paginate/virtualize the island, hydrate with `rootMargin` earlier + render skeleton; or SSR the first page of comments as HTML and hydrate controls only.

## MENTAL MODEL — Performance

> Performance is **spending**: bytes, CPU, and requests, per stage of the load. Static HTML is revenue; every island is an expense with an interaction as its return. Write the budget, measure the spend, and cut features whose return doesn't clear their cost.

## Official Documentation

- Rendering & performance concepts: https://docs.astro.build/en/concepts/why-astro/
- Images: https://docs.astro.build/en/guides/images/
- Fonts: https://docs.astro.build/en/guides/fonts/
- Web Vitals: https://web.dev/articles/vitals

## What I Should Know Before Continuing

1. The eight-stage load model. Which stage does each Core Web Vital live in?
2. Write your page-category JS budgets from memory.
3. Why is hydration CPU often worse than download bytes?
4. What would you measure first on a slow `/blog/[slug]` page?
