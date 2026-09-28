# Module 09 — Framework Interop, React Islands & Astro vs React

> Phase 8. Astro is **not** a replacement for React — and React is not the architecture of an Astro site. Different tools, different jobs.

## Part 1 — Astro vs React (dedicated comparison)

### Component model

| | **Astro component** | **React component** |
|---|---|---|
| File | `.astro` (frontmatter + HTML template) | `.tsx` (JSX function) |
| Runs | server/build only | server (SSR) **and** browser (hydration) |
| Output | HTML string | vdom → DOM |
| Exists in browser | ❌ never | ✅ after hydration |
| Re-render | none (rebuild/request) | state/prop changes |
| JS shipped | 0 | runtime + component |

### Props

| | Astro props | React props |
|---|---|---|
| Typing | `interface Props` (compile-time) | TS props types (compile-time) |
| Updates | re-rendered per request/build | reactive — triggers re-render |
| Functions | allowed (server-only) | first-class (callbacks) |
| Serializable boundary | must be serializable to cross into islands | N/A inside React tree |

### State

| | Astro | React |
|---|---|---|
| Component state | **doesn't exist** | `useState`, `useReducer`, refs |
| Shared state | URL, DOM, server session/DB, tiny stores | Context, stores |
| Data loading | frontmatter `await` (build/request) | RSC/`use`/`useEffect`/query libs |
| Mental model | state is owned by **URL/server** | state is owned by **components** |

### Rendering

| | Astro | React (client-rendered SPA) |
|---|---|---|
| First paint | HTML already complete | after JS boot + render |
| SEO | native (it's HTML) | fine with SSR/RSC setups; fragile CSR-only |
| Interactivity | opt-in islands | default everywhere |
| Routing | real navigations (or ClientRouter enhancement) | client router |

### Where React is genuinely great INSIDE Astro

1. **Stateful widgets** — multi-step forms with derived state, sortable/filterable tables, tree views.
2. **Rich editing** — editors, whiteboards, spreadsheets.
3. **Heavy ecosystem** — you need a specific React-only library (dnd-kit, a chart lib, TipTap).
4. **App zones** — the authenticated dashboard can legitimately be one large React island (an "app island") while marketing/docs stay Astro. Even then, keep pages SSR-rendered where possible and isolate by route.

**Where React is waste inside Astro:** static content, simple toggles, forms that POST, navigation, anything in the escalation ladder below rung 6.

## Part 2 — Multiple UI frameworks: evaluation

Official integrations (verify majors at [integrations directory](https://astro.build/integrations/)): `@astrojs/react`, `@astrojs/vue`, `@astrojs/svelte`, `@astrojs/solid-js`, `@astrojs/preact` (+ community e.g. Lit).

| Framework | Island size (runtime) | Strengths as islands | Watch out |
|---|---|---|---|
| **React** | medium-large | ecosystem, hiring, libs | runtime cost; temptation to SPA-ify |
| **Preact** | small | React-like, tiny | smaller ecosystem; aliasing quirks |
| **Svelte** | small | compile-away runtime, great DX | smaller job market |
| **Vue** | medium | gentle learning curve, good libs | runtime cost |
| **Solid** | smallest reactive | fine-grained reactivity, perf | smaller ecosystem |

**Course standard: ONE framework per project — React.** Reasons: ecosystem depth for the full-stack SaaS project (forms, tables, auth UI), the Astro↔React pattern documentation, and your stated React learning path. The *architecture* doesn't depend on the choice — boundaries, directives, and budgets do.

### Why mixing frameworks is a real cost (not a flex)

- Each framework adds its **own runtime chunk** (React + Vue + Svelte = three downloads if all hydrate).
- **No shared state** between framework islands — cross-framework communication is URL/events/vanilla modules only.
- Testing, design systems, and hiring multiply.
- Two frameworks = two hydration schedules and two sets of a11y pitfalls.

**Acceptable mix** (justify in writing): e.g. legacy Vue map widget + new React app area — with a documented boundary (separate routes) and a migration plan.

## Part 3 — Framework boundaries, concretely

```text
src/components/            ← Astro components (documents)
src/components/react/      ← React islands ONLY
```

Rules:
1. Islands import React libraries; Astro components import islands **as tags with directives**.
2. Props across the boundary are **serializable** (strings, numbers, plain JSON).
3. No React component may import `src/server/**`.
4. State crossing islands: URL / events / shared vanilla module (Module 24).
5. Design system: Astro Button for docs; React Button inside app islands (Module 06).

## Code: the canonical boundary

```astro
---
// src/pages/app/dashboard.astro — SSR page, Astro shell
export const prerender = false;
import BaseLayout from '../../layouts/BaseLayout.astro';
import DashboardFilters from '../../components/react/DashboardFilters';
import RevenueChart from '../../components/react/RevenueChart';
const user = Astro.locals.user!;
const initialData = await loadDashboard(user.id); // server-side fetch
---

<BaseLayout title="Dashboard">
  <h1>Hi {user.name}</h1>
  <DashboardFilters client:load initialFilters={Astro.url.searchParams.get('filter')} />
  <RevenueChart client:visible data={initialData.series} />
</BaseLayout>
```

Everything outside those two tags is free.

## Common Mistakes

**BAD:** "We'll use Svelte for the site and React for the dashboard and Vue for the docs widget."

**GOOD:** one framework; if legacy forces a second, isolate by route and plan the exit.

**BAD:** recreating Next.js App Router behavior with a global React root and client-side routing.

**GOOD:** if the product is fundamentally an SPA, evaluate Module 40 honestly instead of forcing Astro.

**BAD:** `useEffect(() => fetch('/api/me'))` to render the user's name in a header island.

**GOOD:** server renders the name in Astro HTML; islands get identity via a small JSON endpoint *only if they need it dynamically*.

**BAD:** passing `Astro` or `locals` objects into islands.

**GOOD:** extract serializable data (`user: { id, name }`).

## Security Notes

- Island props are visible in HTML/JS — never send server-only session internals.
- Cross-framework message buses must validate event payloads (untrusted DOM events).
- UI framework "protected routes" are cosmetic; enforcement is server-side (Modules 21–23).

## Performance Notes

- React runtime ~40+ KB gzip (React 19, min+gzip ballpark) — shared across React islands; Svelte/Solid islands can be cheaper standalone. Numbers vary by version: **measure your own build**.
- A Preact island can be a smart budget play for tiny widgets if you're not otherwise shipping React.
- Count runtimes per page: ideal is 0, acceptable is 1.

## Exercises

**Beginner.** Build the same "interactive star rating" twice: pure Astro+script and React island. Compare bytes and lines of code. Write which you'd ship and why.

**Intermediate.** Dashboard page with two React islands sharing filter state via URL (`?range=30d`); both react to `popstate`.

**Production.** Add a Svelte island to a React project *deliberately* (documented legacy reason). Measure the page's JS with and without it; write the migration note.

**Architecture Challenge.** Team wants: "Astro + React marketing, Vue docs widgets, Svelte admin." Cost it: runtimes, state, hiring, testing. Propose the single-framework alternative and the one exception you'd allow.

> **Review:** three runtimes on three sections = three testing matrices and zero shared component logic; cross-cutting state must leave the frameworks anyway (URL/server). One framework (React) everywhere islands exist; Vue widget stays only if rewrite cost exceeds 2 quarters — with a route-level boundary and no new Vue.

**Debugging Challenge.** Vue island renders HTML but never hydrates; console: "Hydration completed but contains mismatches" or nothing at all. Cause: the component was imported in `.astro` but rendered via `components` prop in MDX where directives don't work. Fix: wrap in an `.astro` component with the directive.

## MENTAL MODEL — Framework Interop

> A UI framework in Astro is a **guest with a leash**: it lives inside islands, gets serializable food, and walks only when a directive says so. React and Astro are complementary — React composes *behavior*, Astro composes *documents*. Choose frameworks for the widget, never for the website.

## Official Documentation

- Framework components: https://docs.astro.build/en/guides/framework-components/
- React integration: https://docs.astro.build/en/guides/integrations-guide/react/
- Integrations directory: https://astro.build/integrations/
- Shared state recipes: https://docs.astro.build/en/recipes/sharing-state/

## What I Should Know Before Continuing

1. Five concrete differences between Astro and React components.
2. Name two cases where React inside Astro is justified — and two where it's waste.
3. What breaks when two island frameworks share state? What survives?
4. Why is `client:only` + multi-framework the worst combination?
