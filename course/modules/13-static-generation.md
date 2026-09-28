# Module 13 — Static Generation

> Phase 11. Astro's default mode and its superpower: build once, deploy HTML to a CDN, sleep at night.

## Concept: what `astro build` produces (static)

```text
astro build
  ├── every route with prerender=true (default in output:'static')
  │     frontmatter runs ONCE per route → HTML file
  └── dist/
        index.html
        blog/my-post/index.html
        _astro/*.css, *.js          (only island/script chunks)
        assets/*                     (optimized images)
        sitemap-index.xml, rss.xml   (if integrations added)
```

Request flow:

```text
Browser → CDN edge → (cache HIT) → HTML bytes → paint
                   → (miss) → origin static file
```

No runtime server. No cold starts. No database at 3 AM. **This is the ideal for content.**

## Prerendering mechanics

| Concept | Behavior |
|---|---|
| `output: 'static'` (default) | all pages prerendered |
| `export const prerender = false` | this page becomes on-demand (**requires an adapter**) |
| `getStaticPaths()` | enumerates dynamic routes at build |
| `Astro.request` / cookies | **not available** on prerendered pages (no request!) |

```astro
---
// src/pages/products/[sku].astro
import { getCollection } from 'astro:content';
export async function getStaticPaths() {
  const products = await getCollection('products');
  return products.map((p) => ({ params: { sku: p.id }, props: { product: p } }));
}
const { product } = Astro.props;
---
```

## Static assets vs processed assets

| Location | Behavior |
|---|---|
| `public/` | copied as-is — favicon, `robots.txt`, `_headers`, downloads |
| `src/assets/` + `astro:assets` | optimized (format/size), hashed filenames |
| imports in frontmatter | bundled/hashed by Vite |

## When static generation is ideal

- Blogs, docs, marketing, portfolios, news archives, changelogs
- Content updated on a **deploy cadence** (minutes-hours-days), not per-second
- SEO-critical pages (fast TTFB globally)
- Sites that must survive traffic spikes — CDN absorbs everything

## When it isn't (preview of Module 14)

- Per-user content, carts, dashboards → SSR
- Prices/inventory changing continuously → SSR or live collections + caching
- Huge page counts (100k+ routes) → hybrid strategies (build core, on-demand the long tail)

## Build performance notes

- Build time ≈ per-page render cost + content processing. Keep `getStaticPaths` data precomputed.
- Images dominate heavy builds — cache `node_modules/.astro`/build cache in CI.
- v7's Rust compiler + Rolldown materially improve build times vs older majors.
- Consider incremental patterns: on-demand rendering of long-tail routes instead of 200k static files (deploy limits exist per platform).

## Common Mistakes

**BAD:** `await fetch('https://api...')` at build for data that changes hourly, with no rebuild trigger.

**GOOD:** webhooks → rebuild (Module 37), or make that route SSR/cached (Module 34).

**BAD:** cookies/`Astro.session` on prerendered pages.

**GOOD:** static page + `server:defer` island (needs adapter) or client fetch to an endpoint for user-specific bits.

**BAD:** 3MB unoptimized hero images in `public/` "because static is fast."

**GOOD:** `astro:assets` `<Image>` with proper sizes (Module 25).

**BAD:** query params driving prerendered pages (`?page=2` on a static page).

**GOOD:** real pagination routes (`/blog/page/2`) or an island that filters client-side for small data sets.

## Security Notes

- Static HTML is **public forever** — nothing private may be baked into it (drafts, internal URLs, hidden product data).
- `public/` is world-readable; never drop exports/CSVs there.
- Build-time env vars with `PUBLIC_` ship to everyone.

## Performance Notes

- Static = best possible TTFB + full CDN cacheability (`max-age` at will).
- Per-route CSS/JS stays tiny because only islands ship JS.
- HTTP caching is trivial: immutable hashed assets (`/_astro/`) + short HTML TTLs if you re-deploy often.

## Exercises

**Beginner.** Static 5-page site + `getStaticPaths` from JSON; inspect `dist/` structure.

**Intermediate.** Blog with draft filtering, date-sorted index, per-post pages; verify zero JS in built HTML except one island.

**Production.** Add sitemap + RSS (Module 26 patterns) and set immutable caching headers for `/_astro/*` via platform config (`public/_headers` or adapter settings).

**Architecture Challenge.** 80k product pages, nightly price updates. All-static build takes 55 minutes. Options? Compare with: *prerender top-N by traffic + SSR/cache the tail; or on-demand with `routeRules` SWR; or scheduled rebuilds if prices are actually nightly. "Static at all costs" is not an architecture.*

**Debugging Challenge.** A page errors at build: `Astro.cookies` is undefined, but works in dev. You used `Astro.cookies` on a prerendered page. Fix: `export const prerender = false` (with adapter) or remove cookie usage; understand dev always emulates requests.

## MENTAL MODEL — Static Generation

> Static generation **moves work to deploy time** and deletes the runtime. Every prerendered page is a promise that its content can be frozen at build. Keep that promise, or switch that one route's rendering strategy — mixing is supported, and it's the point.

## Official Documentation

- Static rendering: https://docs.astro.build/en/basics/rendering-modes/
- getStaticPaths: https://docs.astro.build/en/reference/api-reference/#getstaticpaths
- Deploy guide: https://docs.astro.build/en/guides/deploy/

## What I Should Know Before Continuing

1. What exists in `dist/` after a static build? Where do islands' chunks live?
2. When is `getStaticPaths` required?
3. Name three things impossible on a prerendered page and their modern alternatives.
4. How do you mix one SSR page into a static site?
