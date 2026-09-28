# Module 08 — Client Directives (Very Deep)

> Phase 7. Every `client:*` directive is a **performance contract**: *what* JS is sent, *when* it runs, and *why*. Verified against the [template directives reference](https://docs.astro.build/en/reference/directives-reference/).

## The contract of a client directive

A UI-framework component **without** a `client:*` directive renders **pure HTML — no JS at all** (and cannot respond to events). Adding a directive:

1. includes the component's JS chunk (and framework chunk) in the page's module graph,
2. schedules **hydration** at a moment the directive defines,
3. the component's `export` becomes interactive after hydration.

**Compile-time rule:** directives must be **statically visible** on a directly imported component. `<X {...attr} />` spreads or dynamic tags do not carry directives.

## The five directives — WHAT / WHEN / WHY / COST

### `client:load` — hydrate immediately
```astro
<BuyButton client:load />
```
- **WHAT:** JS downloaded and executed as soon as possible (high priority).
- **WHEN:** on page load (module scripts; not waiting for idle/visibility).
- **WHY:** immediately-visible, mission-critical interaction (buy button, header cart, primary form with complex live validation).
- **COST:** competes with LCP resources and input readiness. Use sparingly — count these per page.

### `client:idle` — hydrate when the browser is idle
```astro
<ChatWidget client:idle />
<ShowHideButton client:idle={{ timeout: 500 }} />
```
- **WHAT:** medium priority; waits for `requestIdleCallback` (or `load` fallback).
- **WHEN:** after initial load settles; `timeout` (added 4.15) forces hydration within N ms.
- **WHY:** useful but not urgent (chat widget, secondary toolbar, analytics-y widgets).
- **COST:** near-zero impact on first load; total bytes still ship (just later).

### `client:visible` — hydrate when scrolled into view
```astro
<HeavyImageCarousel client:visible />
<Comments client:visible={{ rootMargin: '200px' }} />
```
- **WHAT:** low priority; `IntersectionObserver` triggers download+hydration.
- **WHEN:** when the element enters (or nears, with `rootMargin`) the viewport.
- **WHY:** below-the-fold heavy content (charts, carousels, comments). If the user never scrolls, you never pay.
- **COST:** hydration jank if the component mounts large trees right as the user arrives; `rootMargin` pre-warms earlier and reduces CLS/INP spikes.

### `client:media="query"` — hydrate when a media query matches
```astro
<SidebarToggle client:media="(max-width: 50em)" />
```
- **WHAT:** hydrates when the media query becomes true.
- **WHEN:** matchMedia change/load.
- **WHY:** device-specific UI (mobile drawer toggle, desktop-only dock). Desktop never downloads the mobile widget's JS.
- **COST:** ~0 for non-matching devices. Watch hydration on resize flip-flop (component stays hydrated once loaded).

### `client:only="framework"` — skip server render, client-only
```astro
<SomeReactComponent client:only="react">
  <div slot="fallback">Loading…</div>
</SomeReactComponent>
```
- **WHAT:** **no HTML from the component at all** — placeholder + immediate client render.
- **WHEN:** like `client:load`, but there's nothing to hydrate; it renders fresh.
- **WHY:** components that **cannot** render on the server (browser-only APIs in render path, `window` access at init, some map/canvas libs).
- **COST:** worst of all: empty first paint region + full client render. You **must** name the framework (`"react"`, `"vue"`, `"svelte"`, `"solid-js"`, `"preact"`) because Astro never sees the component. Always give `slot="fallback"`.

### Priority ladder (memorize)

```text
load  →  idle  →  visible  →  media  →  only
most urgent, most expensive ──────────────► most deferred / most degraded
```

## What JavaScript gets sent?

For any directive: the **component chunk** + its imports + the **shared framework runtime**. The directive itself is a tiny inline loader script (different per directive) that decides when to `import()` the chunk. `client:visible`/`media` include tiny observers (~1 KB). `client:only` includes **no SSR HTML**, so the framework must do first render in the browser too.

## Decision table (real scenarios)

| Widget | Directive | State owner | Notes |
|---|---|---|---|
| Mobile nav toggle | **none** — vanilla script or `<details>` | DOM | framework is overkill |
| Buy button on product page | `client:load` | server (cart Action) | critical path |
| Chat widget | `client:idle` | island-local + server | never blocks LCP |
| Testimonial carousel | `client:visible` | island-local | or CSS scroll-snap, 0 JS |
| Analytics dashboard filters | `client:load` (in-app page) | URL + server | app page, above fold |
| Mobile-only filter drawer | `client:media="(max-width: 50em)"` | island-local | desktop pays 0 |
| Map with WebGL init | `client:only="react"` + fallback | island-local | browser-only render |

## Custom directives (advanced, official)

Integrations can add directives (`addClientDirective` in the [Integrations API](https://docs.astro.build/en/reference/integrations-reference/)) — e.g. community `client:click`, `client:postpone`. Only build one after exhausting the five.

## Script & style directives (same family)

| Directive | Effect |
|---|---|
| `is:inline` | raw script/style in HTML: no bundling, no dedupe, no TS/imports |
| `define:vars` | JSON-serializes frontmatter values **into** the script (public data!) |
| `is:global` | unscoped styles |
| `is:raw` | treat children as text |

## Common Mistakes

**BAD:** `client:load` on everything "so it works."

**GOOD:** no directive until interaction proves it; then the *laziest* directive that works.

**BAD:** `client:only` to dodge an SSR bug (e.g. `window` access in module scope).

**GOOD:** fix the SSR-unsafe code (guard with `typeof window`), keep server render + `client:visible`. `client:only` is a last resort, not a bugfix.

**BAD:** hydrating on `visible` a component whose *loading state* causes layout shift.

**GOOD:** reserve space (CSS aspect-ratio/min-height) + optional `rootMargin` warm-up.

**BAD:** forgetting that `client:idle` still downloads the bytes — idle is timing, not diet.

**GOOD:** if the widget is optional, `visible`/`media` avoid the bytes entirely.

**BAD:** directives on `mdx` `components` prop-passed components (not supported — directive must be in `.astro` directly).

**GOOD:** create a tiny `.astro` wrapper that imports the component and adds the directive.

## Security Notes

- Directives control **when** public JS runs — never put secrets in island code or `define:vars`.
- `client:only` components run only in the browser: do not trust their "hidden" UI as access control (Module 22).

## Performance Notes

- **Hydration budget rule:** ≤ 1 `client:load` above the fold per page; everything else idle/visible/media.
- React islands share one runtime; a second React island is mostly the component chunk.
- Prefer `client:visible={{ rootMargin: '200px' }}` for below-fold interactive regions users will hit (reduces late-INP jank).
- Audit quarterly: grep `client:` across `src/`; every hit must map to a documented interaction.

## Exercises

**Beginner.** Same `<Counter />` React component with `client:load`, `client:idle`, `client:visible`. Compare Network tab start times; write one sentence each.

**Intermediate.** Build a pricing calculator island (`client:load`, URL-synced sliders) + a testimonial carousel (`client:visible`). Total page JS under a 60 KB budget (measure!).

**Production.** Mobile drawer with `client:media="(max-width: 50em)"`; verify desktop Network tab downloads **zero** island JS; resize to mobile and confirm hydration.

**Architecture Challenge.** "Homepage with hero form (email), 3 feature cards, a demo video embed, a pricing table, and a support chat." Write the directive plan with bytes estimate.

> **Review:** hero form: native form + optional 2 KB vanilla script → 0 framework JS; cards/table: Astro; video embed: facade thumbnail + click-to-load iframe (privacy + perf); chat: `client:idle`. Framework islands: possibly none at all.

**Debugging Challenge.** `<Chart client:visible />` never becomes interactive. DevTools shows the chunk never requested. The page uses `<ClientRouter />`, navigated via client-side nav, and the chart is re-rendered but the observer was bound at first `astro:page-load` only. Fix: re-init on `astro:page-load` / rely on island lifecycle (islands should self-manage; plain scripts must listen to `astro:page-load`).

## MENTAL MODEL — Directives

> A directive is a **scheduling decision over a cost you already accepted**. `load/idle/visible/media` control *when*; `only` also degrades *what*. Choose the laziest schedule that preserves the interaction the user actually came for, and let everything else be HTML.

## Official Documentation

- Directives reference: https://docs.astro.build/en/reference/directives-reference/
- Framework components: https://docs.astro.build/en/guides/framework-components/
- Client-side scripts: https://docs.astro.build/en/guides/client-side-scripts/

## What I Should Know Before Continuing

1. Order the five directives by priority and by typical bytes impact.
2. What extra requirement does `client:only` have, and why is its first paint worse?
3. Why can't a directive live behind a spread attribute?
4. When is `rootMargin` on `client:visible` the right call?
