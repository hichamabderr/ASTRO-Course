# Module 14 — SSR, Adapters & Rendering Strategy

> Phases 12, 18, 19 of the plan. On-demand rendering: adapters, per-page `prerender`, and the decision framework STATIC vs SSR vs MIXED.

## Concept: on-demand rendering

```text
Browser → CDN → adapter runtime (Node/Workers/Functions)
            → middleware → page frontmatter runs PER REQUEST
            → HTML response (optionally cached — Module 34)
```

An **adapter** is an Astro integration that knows how to (a) build your server routes for a specific runtime and (b) run them there.

## The two modes (Astro 5+; `hybrid` is gone ⛔)

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  output: 'server',              // or omit for 'static' default
  adapter: node({ mode: 'standalone' }),  // 'middleware' for embedding in Express etc.
});
```

| Mode | Page default | Opt-out |
|---|---|---|
| `output: 'static'` | prerendered | `export const prerender = false` (adapter required) |
| `output: 'server'` | on-demand | `export const prerender = true` |

**Mixed is normal:** marketing static + account SSR in one app. Rendering is a **per-page property**.

## What SSR unlocks (and costs)

| Unlocks | Costs |
|---|---|
| cookies, sessions, auth | per-request CPU (cold starts on serverless) |
| fresh data (DB/API) | origin latency + DB load |
| request-derived HTML (geo, A/B, personalization) | caching complexity |
| Actions/endpoints at runtime | ops: logs, env, scaling |

## Rendering strategy decision framework

For **every page**, ask:

1. Does it need **runtime data** (per-request)?
2. Does it use **cookies/session**?
3. Does it depend on **authenticated users**?
4. Does it **change frequently** (more often than your deploy cadence)?
5. Can it be **built once** and remain honest?

```text
All answers "no"  → STATIC (prerender)
Any "yes"         → candidate for SSR… then ask:
   Can the DYNAMIC part be an island/server island + cache the rest? → static shell + deferred fragment
   No                                                                  → SSR page (with caching — Module 34)
```

### The capstone pattern (memorize)

| Route | Strategy | Why |
|---|---|---|
| Landing, pricing, docs, blog | static | content, deploy cadence |
| Account dashboard, settings, billing | SSR | session-bound |
| Admin | SSR + strict authz | privileged, dynamic |
| Product page (static shell) + live price | static + `server:defer` | best of both |
| Cart | island + Actions/session | stateful interaction |
| Search | island + endpoint/SSR | query-time data |

## Adapters (official, Sep 2026)

| Adapter | Runtime | Notes |
|---|---|---|
| `@astrojs/node` | Node.js 22+ | `standalone` server or `middleware` mode; Docker-friendly |
| `@astrojs/vercel` | Vercel functions/edge | platform features (ISR interplay, image service) |
| `@astrojs/netlify` | Netlify functions/edge | auto config |
| `@astrojs/cloudflare` | Workers (workerd) | notable v6 breaking changes; growing importance post-acquisition |

Community adapters exist (e.g. Deno, Appwrite) — evaluate maintenance before adopting.

**Cloudflare caveat (2026):** Astro's dev server now runs on workerd (v6+), and the Cloudflare adapter changed significantly at v6. If you deploy there, read its changelog. Edge runtimes restrict Node APIs and some DB drivers (Module 37).

## Sessions + SSR

Sessions (stable) require on-demand rendering + a storage driver. Node/Cloudflare/Netlify adapters configure defaults; others set `session.driver` (unstorage). On prerendered pages `Astro.session` is `undefined` (with a logged error).

## Common Mistakes

**BAD:** `output: 'server'` for a 90% marketing site "for flexibility."

**GOOD:** static default; opt pages into SSR. Crawlability & TTFB stay maximal.

**BAD:** SSR pages that could be static + webhooks.

**GOOD:** use the decision framework per route; revisit quarterly.

**BAD:** reading `Astro.cookies.get('theme')` in the shared layout of a static site.

**GOOD:** theme via `localStorage` + tiny script (client concern), or SSR layout with `server:defer` fragments for personalized chrome.

**BAD:** assuming SSR means "uncacheable."

**GOOD:** `routeRules` / `Astro.cache` (Module 34) make SSR cheap.

## Security Notes

- SSR exposes server code to request input at scale — validate everything (`Astro.params`, headers, bodies).
- Prerendered pages must never embed request-derived secrets (there is no request).
- Choose `SameSite`/`Secure` cookie settings per environment (Modules 21/23).

## Performance Notes

- Cold starts: keep server bundles lean (avoid giant SDKs in request paths); prefer connection pooling (Module 19).
- Static shell + deferred dynamic fragments often beats full SSR on LCP.
- Cache aggressively (Module 34): SSR ≠ slow.

## Exercises

**Beginner.** Convert a static page to `prerender = false` with the Node adapter; log `Astro.request.headers` to prove per-request rendering.

**Intermediate.** Mixed site: 3 static pages + `/account` SSR reading a cookie "user". Verify `dist/client` vs `dist/server` output.

**Production.** Apply the decision framework to 10 real routes of a site you know; write a table with justifications; implement one static→"static + server:defer" upgrade.

**Architecture Challenge.** E-commerce PDP: catalog info daily, price hourly, stock per-minute, recommendations per-user. Design the rendering strategy layer by layer.

> **Review:** static shell (SEO content, images) + `server:defer` price/stock fragments (cached 60s by tag) + recommendations island (`client:visible`, personalized fetch) + cart island. Full SSR only for checkout/account.

**Debugging Challenge.** `astro build` succeeds but deploying static output, `/account` shows a 404 from the host. You set `prerender = false` on a `output: 'static'` project **without an adapter** — dev hid the problem. Add the adapter (or make the page static) and recheck `dist/server/` exists.

## MENTAL MODEL — Rendering Strategy

> Rendering is a **per-route budget**: freeze what can be frozen, compute what must be computed, defer what is slow, cache what repeats. "Static vs SSR" is not an identity — it's a spreadsheet of per-page answers to five questions.

## Official Documentation

- Rendering modes: https://docs.astro.build/en/basics/rendering-modes/
- Adapter guide: https://docs.astro.build/en/guides/integrations-guide/#official-integrations
- Adapter API: https://docs.astro.build/en/reference/adapter-reference/
- Deploy: https://docs.astro.build/en/guides/deploy/
- Sessions: https://docs.astro.build/en/guides/sessions/

## What I Should Know Before Continuing

1. The five decision questions — and what each "yes" implies.
2. `output: 'static'` vs `'server'`: page defaults and opt-outs.
3. Which adapters are official? What's special about Cloudflare in 2026?
4. Design the rendering strategy for a "blog + dashboard" app in one table.
