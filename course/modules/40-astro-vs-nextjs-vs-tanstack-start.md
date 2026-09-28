# Module 40 — Astro vs Next.js vs TanStack Start

> Phase 98. A detailed, non-marketing comparison. **No winner is declared.** The goal: knowing *when each architecture makes sense*.

## The contenders (Sep 2026 context)

| | **Astro 7** | **Next.js** (15/16 era) | **TanStack Start** |
|---|---|---|---|
| Core metaphor | Content + selective islands | Full-stack React (RSC + router) | Full-stack React + TanStack Router |
| Default JS shipped | ~0 | React runtime + router | React runtime + router |
| Rendering | static / SSR / server islands per route | SSG/SSR/RSC streaming per segment | SSR + client, route-level |
| Content tooling | **first-class** (Content Layer, MDX, Starlight) | good (MDX), conventions/community | DIY / library choice |
| Server mutations | Actions / endpoints | Server Actions / route handlers | server functions |
| Routing | files in `src/pages/` | app router (files + layouts) | code-first TanStack Router |
| Data loading | frontmatter/getStaticPaths/live collections | loaders/RSC fetch patterns | loaders + TanStack Query |
| Auth ecosystem | library-based (Better Auth etc.) | huge ecosystem | ecosystem via React |
| Ecosystem gravity | content/web platform | largest React meta-ecosystem | TanStack family |
| Owned/hosted context | Cloudflare (2026), framework-agnostic hosts | Vercel-optimized but portable | community/company (TanStack) |

## Feature-by-feature (the decision table)

| Need | Astro | Next.js | TanStack Start |
|---|---|---|---|
| Blog / docs / content platform | **best-in-class** | good | DIY |
| Marketing site + SEO | **best-in-class** | good | good (SSR) |
| E-commerce storefront | excellent (static + islands) | excellent | good |
| Highly interactive dashboard | good (app-zone islands) | **excellent** | **excellent** |
| Real-time collaborative UI | possible but atypical | good | good |
| Mixed static + SaaS in one codebase | **its native shape** | fine | fine |
| Client JS budget (content pages) | **~0** | higher | higher |
| Islands / partial hydration | native | RSC != islands (different model) | client components |
| Client routing polish | opt-in ClientRouter | native | native, fine-grained |
| Server functions | Actions (typed) | Server Actions | server functions |
| Learning curve | HTML-first; low for content | React-ops heavy | router-centric |
| Hiring | TS/web generalists | React specialists | React specialists |

**Model differences that matter:**
- Next.js **RSC** streams server components *inside* a React tree — a different answer to "what ships to the client" than islands, with its own constraints (serialization across the RSC boundary, cache semantics).
- TanStack Start optimizes for **type-safe routing + client data** ergonomics in a React full-stack app.
- Astro optimizes for **documents first**; apps are guests.

## Scenario verdicts

| Scenario | Pick | Why |
|---|---|---|
| Docs + blog + changelog for a product | **Astro** | Content Layer, zero-JS, Starlight |
| Marketing site for a React SaaS | **Astro** | perf/SEO; share UI via islands |
| The React SaaS dashboard itself | **Next.js or TanStack Start** (or Astro app-zone if the team is Astro-first) | client-heavy, router-centric |
| E-commerce storefront + content | **Astro** | static PDPs, SSR checkout routes |
| Real-time whiteboard | **neither Astro-first** — Vite+React/Svelte app | interaction-first |
| Startup "everything app" with small team | depends: docs-heavy → Astro; app-heavy → TanStack Start/Next | dominant workload decides |
| Blog that "might become an app" | **Astro** | islands grow gracefully; migrating docs out of Next later is harder than growing islands |

## Fair criticisms of each

| | Weaknesses (honest) |
|---|---|
| Astro | not for interaction-dominant UIs; islands need explicit state strategy; smaller "app framework" ecosystem than Next |
| Next.js | heavier default JS; complexity of RSC mental model; content tooling is not the center; framework churn |
| TanStack Start | younger ecosystem; you assemble more; less content/SEO scaffolding |

## Migration/porting notes

- Content sites Astro → Next: mostly MDX + routing work; SEO must be rebuilt.
- Next SPA-ish pages → Astro: extract islands per widget; data moves to frontmatter/actions; biggest win is usually deleting client fetch waterfalls.
- Astro app-zone → TanStack Start: components port (React); routing and data layers rewrite.

## MENTAL MODEL — Choosing

> Choose the framework whose **defaults match your product's dominant unit**: documents → Astro; React application surfaces → Next.js/TanStack Start; interaction-first canvas → a client app. The architecture you want to *stop fighting* is the right one.

## Official Documentation / references

- Astro: https://docs.astro.build/ · Next.js: https://nextjs.org/docs · TanStack Start: https://tanstack.com/start
- Astro islands: https://docs.astro.build/en/concepts/islands/
- RSC model: https://react.dev/reference/rsc

## What I Should Know Before Continuing

1. Explain the *model* difference: islands vs RSC vs client-first.
2. For each of five products you know, name the pick + the one sentence why.
3. What are each contender's honest weaknesses?
4. When is the hybrid (Astro shell + app router zone) the right compromise — and what does it cost?
