# Module 39 — Anti-Patterns & When NOT to Use Astro

> Phases 73–75. The "DO NOT DO THIS" compendium — and the honest boundaries of the framework.

## Part 1 — Anti-patterns (by area)

### Architecture
| Anti-pattern | Instead |
|---|---|
| ⛔ Every component is a `client:*` island | static by default; islands earn their hydration |
| ⛔ React for static content | Astro components |
| ⛔ Recreating a SPA inside Astro (one root island + client router) | real routes + islands; or a different framework (Part 2) |
| ⛔ Hydrating "for consistency" | extract the interactive control only |
| ⛔ Multiple UI frameworks casually | one framework; document exceptions |

### Data
| Anti-pattern | Instead |
|---|---|
| ⛔ `useEffect` fetch of server-owned data | frontmatter/actions; props |
| ⛔ Client fetch to your own endpoints for first-party UI | typed actions |
| ⛔ Global client state for everything | HTML/URL/server/island ownership ladder (Module 24) |
| ⛔ State library before state ownership exists | prove the need first |

### Server & security
| Anti-pattern | Instead |
|---|---|
| ⛔ Secrets in islands / `PUBLIC_` misuse | `process.env`, server-only modules |
| ⛔ Middleware as the only authorization layer | re-check in handlers/services |
| ⛔ SQL in page frontmatter | service → repository layering |
| ⛔ Trusting `Astro.params` unvalidated | Zod at boundaries |
| ⛔ `set:html` with user data | escape/sanitize |

### Performance
| Anti-pattern | Instead |
|---|---|
| ⛔ `client:load` everywhere | directive ladder |
| ⛔ Unoptimized images in `public/` | `astro:assets` |
| ⛔ Third-party tag soup | facades, consent, budgets (Modules 29/31) |
| ⛔ Framework widget for `<details>` | platform features (Module 06) |

### Process
| Anti-pattern | Instead |
|---|---|
| ⛔ Copying 2024 tutorials into Astro 7 | verify APIs (orientation 04) |
| ⛔ No `astro check`/tests in CI | quality gates (Modules 02/30) |
| ⛔ "We'll optimize later" without a written budget | budgets first (Module 29) |

## Part 2 — When NOT to use Astro

Astro can host React/Vue/Svelte islands — **but that does not make every SPA architecture a good Astro architecture.** Choose something else when:

1. **The product is fundamentally a highly interactive application** — a Figma-like editor, a real-time game client, a trading terminal. Client rendering *is* the product.
2. **Client state dominates** — complex cross-cutting client state machines, offline-first sync engines. Framework routers + state ecosystems (React + TanStack Router/Query, etc.) are purpose-built.
3. **Full client-side routing is central** — instant view-to-view transitions with shared in-memory state everywhere; persistent audio/video across *every* view with deep linking into app state.
4. **The ecosystem gravity is elsewhere** — a Next.js-only component suite, Vercel-specific features, or a SvelteKit team's expertise is the constraint that matters.
5. **The team is an SPA team** with no appetite for server/content thinking — the model mismatch will produce a bad Astro project *and* a bad app.

**The test:** *How many of the app's routes are documents?* If < ~20% and falling, Astro's advantages (content layer, static-first, zero-JS default) are mostly unused — pick the framework whose defaults match your reality (Module 40).

You *can* build the SaaS dashboard as one big `client:load` island on `/app/*` while marketing/docs stay Astro — that hybrid is legitimate (Module 14's capstone pattern). Just don't lie to yourself about which part is an SPA.

## Part 3 — When Astro is clearly the right call

- Content-heavy sites (blogs, docs, news, marketing, portfolios, course sites).
- Mixed workloads: static content + app zones + occasional SSR.
- SEO/perf budgets that forbid SPA boot costs.
- Teams comfortable with "server renders documents; islands render widgets."

## Common Mistakes (meta)

**BAD:** choosing Astro to "avoid learning a backend," then bolting on client state forever.

**GOOD:** Astro forces the server question — embrace Actions/endpoints (Modules 15–16) or pick a pure-client stack.

**BAD:** choosing Next.js to "have the option of interactivity" for a docs site.

**GOOD:** match the tool to the dominant workload; migration paths exist both ways.

## MENTAL MODEL — Judgment

> Frameworks are **default tradeoffs**. Astro's default is "document-first, JS last." If your product's default is "interaction-first, state everywhere," you'll fight it every day — and the honest senior move is to pick a different default, not to tweet that the framework is bad.

## Official Documentation

- Why Astro: https://docs.astro.build/en/concepts/why-astro/
- Islands: https://docs.astro.build/en/concepts/islands/
- Any-time architecture guidance: https://docs.astro.build/en/concepts/islands/#common-patterns

## What I Should Know Before Continuing

1. Five anti-patterns from memory + their replacements.
2. The "how many routes are documents" test — apply it to a product you know.
3. Design the hybrid (marketing Astro + app SPA-island) and its boundaries.
4. What's the ethical thing to say when a client's app is a bad Astro fit?
