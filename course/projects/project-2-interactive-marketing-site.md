# Project 2 — Interactive Marketing Site ("Astro in Action")

> Modules applied: 06–09, 16–17, 24, 27–29, 31, 35. Deliverable: a marketing site that *feels* app-like while shipping almost no JS.

## Feature list

- Landing page: hero, features, logo wall, testimonials, pricing table, FAQ, CTA
- Pricing page with **interactive plan calculator** (island, URL state)
- Blog roll (reuse Project 1 content layer)
- Newsletter signup: **native form + Action**, progressive enhancement
- Testimonial carousel: `client:visible` island — or CSS scroll-snap (justify your choice in writing!)
- Command-palette-style quick nav (`client:idle` island, keyboard-first, a11y per Module 28)
- View transitions (`<ClientRouter />`) with persisted header + reduced-motion support
- Scroll-reveal animations (CSS/observer, enhanced-only)
- Consent-gated analytics beacon (Module 31)
- A/B-ready hero via `routeRules` caching notes (Module 34)

## Architecture

```text
src/pages/{index,pricing,blog/**}.astro
src/components/react/{PlanCalculator, QuickNav, TestimonialCarousel?}.tsx
src/actions/index.ts        newsletter: accept:'form'
src/middleware.ts           security headers + logging
```

## JS budget (write it in the repo, enforce it)

| Page | Budget |
|---|---|
| Landing | ≤ 60 KB gzip total JS; ≤ 1 `client:load` (calculator only on /pricing) |
| Pricing | calculator island ≤ 30 KB + shared React |
| Blog pages | ~0 (Project 1 rules) |

## Milestones

1. **M1 — Static story:** full landing in Astro + CSS (Module 06) — measure baseline.
2. **M2 — Interactivity:** calculator (URL `?plan=&seats=`), carousel decision documented.
3. **M3 — Forms:** newsletter Action + no-JS path + error/pending a11y (16–17).
4. **M4 — Polish:** ClientRouter + persist + reduced-motion (27).
5. **M5 — Measure:** Lighthouse + WebPageTest; write the "why not a SPA" one-pager (Module 29 case study method).

## Definition of done

- [ ] All interactions keyboard-operable; axe clean
- [ ] Newsletter works with JS disabled (verified with devtools "Disable JavaScript")
- [ ] Budget respected; numbers in `PERFORMANCE.md`
- [ ] View transitions degrade to normal navigation in unsupported browsers
- [ ] Calculator state survives share/reload

## Stretch

- Command palette searches docs + blog (Module 35 index)
- Locale switcher demo (2 locales)
- Route cache experiment: SSR pricing fragment with `swr` (34)

## Architecture Challenge (graded)

> "Add a live chat widget, a video background, and an exit-intent modal — keep the budget."

Expected: chat `client:idle`; video facade + click-to-load; modal as `<dialog>` + tiny script or one `client:visible` island; budget negotiation documented (drop or defer features rather than silently exceed).
