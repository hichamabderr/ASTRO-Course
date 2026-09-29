# Module 27 — View Transitions & Animation

> Phases 22–23. The native View Transition API, Astro's `<ClientRouter />` enhanced navigation — and when *not* to use either.

## Concept: two different things with one name

1. **Native browser View Transitions** — `document.startViewTransition()`: the browser screenshots old state, applies DOM change, animates between. Cross-document transitions also exist natively (CSS-only, MPA) in modern browsers.
2. **Astro's client-side router** — `<ClientRouter />` (formerly `<ViewTransitions />` ⛔ deprecated name) from `astro:transitions`: intercepts same-origin link clicks and back/forward, fetches the next page, swaps the DOM **without a full reload**, optionally animating with the View Transition API.

```astro
---
// src/layouts/BaseLayout.astro
import { ClientRouter } from 'astro:transitions';
---
<html lang="en">
  <head>
    <title>{title}</title>
    <ClientRouter />
  </head>
  …
</html>
```

Default behavior: cross-fade between pages. Fallback: normal full navigation on unsupported browsers (`fallback` prop controls animation fallback).

## Directives for animation

```astro
<header transition:persist>…</header>                 <!-- keep element mounted across navs -->
<h1 transition:animate="slide">…</h1>                  <!-- fade | slide | none | custom -->
<img transition:name="hero" />                          <!-- shared-element morph across pages -->
```

- `transition:persist` — nav bars, audio players, island instances survive navigation.
- `transition:animate` — per-element animations (`none` for scroll containers!).
- `transition:name` — matching names morph between pages (the "hero image flies" effect).
- Respect `prefers-reduced-motion` in custom animations.

## Lifecycle events (v7: use event names, constants were removed ⛔)

```js
document.addEventListener('astro:page-load', initAll);       // every page shown (incl. first)
document.addEventListener('astro:after-swap', syncTheme);    // DOM swapped, before paint
document.addEventListener('astro:before-preparation', (e) => { /* can intercept navigation */ });
```

**The classic bug:** third-party scripts and your own `<script>`s that bind to `DOMContentLoaded` never re-run on client-side navigation. Bind init logic to `astro:page-load`, or use `transition:persist` for stateful widgets. (v7 removed `TRANSITION_*` constants and `createAnimationScope()` — compare `event.type === 'astro:after-swap'`.)

## Tradeoffs — do NOT SPA-ify everything

| ClientRouter gives | ClientRouter costs |
|---|---|
| app-feel transitions | JS always on (router ~2–4 KB gzip — measure) |
| persistent islands across pages | stale-state bugs (third-party scripts, listeners) |
| fewer full reloads | scroll/focus management needs care; URL/SEO unaffected (real URLs kept) |

**Use it:** marketing sites, docs, blogs where polish matters.
**Skip it:** mega-form flows (full reloads are refreshing), or when debugging costs outweigh polish. Native CSS `@view-transition { navigation: auto; }` (cross-document, no router) is a lighter alternative — verify current browser support before relying on it.

## Animation toolkit ladder (Module 06's philosophy applied)

```text
1. CSS transitions/animations            hover, focus, dialogs
2. View Transitions (native/Astro)       page navigation polish
3. Tiny <script> + WAAPI/observer        scroll reveals
4. Motion One / motion (library)         complex sequences — islands only if stateful
5. Framework animation libs              only inside islands already justified
```

Reduced motion:

```css
@media (prefers-reduced-motion: reduce) {
  * { animation-duration: 0.01ms !important; transition-duration: 0.01ms !important; }
}
```

## Common Mistakes

**BAD:** `transition:persist` on everything → stale DOM vs new page expectations.

**GOOD:** persist only chrome (nav, player) with explicit re-init on `astro:page-load`.

**BAD:** heavy FLIP animation library for a menu.

**GOOD:** CSS transform animation.

**BAD:** custom animations ignoring `prefers-reduced-motion`.

**GOOD:** media-query reduced motion (a11y requirement, not a nicety).

**BAD:** scroll-jacking reveals that hide content from screen readers or without JS.

**GOOD:** content visible by default; animation is enhancement.

## Security Notes

- ClientRouter fetches pages with credentials same-origin — ensure CSRF protections still cover POSTs (routers don't change the threat model).
- Persisted islands keep state across navigations — don't persist sensitive UI state beyond need.

## Performance Notes

- Router JS is small but nonzero — budget it; use site-wide only if it earns its keep.
- `transition:persist` keeps island instances alive → memory retained; intentional.
- Animations on compositor properties only (`transform`, `opacity`) for 60fps.

## Exercises

**Beginner.** Add `<ClientRouter />`; change default fade to `slide` on `h1`; persist the header.

**Intermediate.** Shared-element morph: blog card thumbnail → post hero (`transition:name`); verify with reduced-motion on.

**Production.** Docs site with persistent sidebar; fix a nav script that breaks after first navigation (the `astro:page-load` bug); add CSS-only `@view-transition` experiment behind a flag.

**Architecture Challenge.** Multi-step checkout wizard + marketing pages. Where do view transitions help or hurt?

> **Review:** marketing/docs: ClientRouter polish; wizard: **no router** — full page loads per step are clearer for focus/scroll/back-button; wizard steps use real URLs/POSTs. Or isolate router to marketing layout only (it's opt-in per layout).

**Debugging Challenge.** Google Analytics only fires on first page. Page-view listener bound to `DOMContentLoaded`. Fix: listen `astro:page-load`, or use a SPA-aware analytics API pattern.

## MENTAL MODEL — Transitions

> `<ClientRouter />` is an **optional enhancement of navigation**, not the navigation itself — URLs, HTML documents, and crawlers still see a normal site. Add it for polish, persist the chrome, re-init on `astro:page-load`, and always leave the reduced-motion escape hatch open.

## Official Documentation

- View transitions: https://docs.astro.build/en/guides/view-transitions/
- astro:transitions API: https://docs.astro.build/en/reference/modules/astro-transitions/
- Directives: https://docs.astro.build/en/reference/directives-reference/

## What I Should Know Before Continuing

1. Native view transitions vs `<ClientRouter />` — what each does.
2. Which lifecycle event re-initializes scripts after navigation?
3. What does `transition:persist` cost and when is it right?
4. Why might a checkout wizard deliberately skip the client router?
