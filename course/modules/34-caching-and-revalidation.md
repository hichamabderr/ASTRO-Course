# Module 34 — Caching & Revalidation

> Phases 56–57. Astro **7** stabilized a first-class route caching system. This module covers the whole stack: browser, CDN, route cache, server, DB — and who invalidates what.

## The four questions (for every cache)

> **WHAT** is cached? **WHERE**? **HOW LONG**? **WHO invalidates it**?

| Layer | What | Where | How long | Invalidation |
|---|---|---|---|---|
| Browser | assets, HTML | user device | `Cache-Control` | versioned URLs (hashes) |
| CDN | static HTML/assets | edge | long (assets) / short (HTML) | deploy purge / tags |
| Route cache (v7) | SSR responses | provider (memory/CDN/…) | `maxAge`/`swr` | `cache.invalidate()` tags/paths |
| Server memory | derived data, sessions | process | seconds–minutes | code/TTL |
| DB | query results | DB buffer/Redis | case-by-case | explicit |

## Static (the easy mode)

- `/_astro/*` hashed → `Cache-Control: public, max-age=31536000, immutable`.
- HTML files → short TTL or `must-revalidate` since redeploys change them.
- Platform config via `public/_headers` (Netlify/Cloudflare), adapter options, or CDN rules.

## Route caching (Astro 7, stable — verified API)

```js
// astro.config.mjs
import { defineConfig, memoryCache } from 'astro/config';
import node from '@astrojs/node';

export default defineConfig({
  adapter: node({ mode: 'standalone' }),
  cache: { provider: memoryCache() },   // adapters/CDN providers may set defaults; 🧪 CDN providers experimental
  routeRules: {
    '/api/[...path]': { swr: 600 },
    '/products/[...slug]': { maxAge: 3600, tags: ['products'] },
    '/blog/[...slug]': { maxAge: 300, swr: 60 },
  },
});
```

Per-request control:

```astro
---
// src/pages/products/[id].astro
export const prerender = false;           // caching applies to on-demand routes
const tags = await getProductTags(Astro.params.id!);
Astro.cache.set({ maxAge: 3600, swr: 60, tags });
const product = await getProduct(Astro.params.id!);
---
<h1>{product.name}</h1>
```

- Endpoints/middleware: `context.cache.set(...)`; read accumulated options via `cache.options`.
- **Merge semantics:** scalars last-write-wins; `tags` accumulate.
- **Invalidation:** tag-based and path-based via `cache.invalidate()` (e.g. after an Action writes data).
- Live collections can return `cacheHint` to feed these settings (Module 10).

```ts
// in an Action handler after a product update:
await Astro.cache.invalidate({ tags: ['products'] });   // check current signature in docs
```

## SSR caching discipline

| Response | Cache? |
|---|---|
| Anonymous product page | ✅ `maxAge` + `swr`, tag `products` |
| Logged-in dashboard | ❌ `private, no-store` |
| Personalized fragment | ❌ (or per-user key — rarely worth it) |
| Public API list | ✅ short `maxAge` |

**Vary carefully:** cookie-varied caches explode key cardinality; prefer `no-store` for logged-in.

## Data revalidation patterns (no invented APIs)

Astro has no magic ISR. The verified toolkit:

1. **Rebuild** — content change → webhook → `astro build` + deploy (static truth).
2. **Short SSR TTLs + SWR** — `routeRules` above; freshness bounded by `swr`.
3. **Tag invalidation** — Actions invalidate cache tags on writes.
4. **Live collections** — request-time data with loader-level caching hints.
5. **Client refresh** — polling islands (Module 24) for volatile widgets.

Pick per data volatility (minutes vs deploys vs per-user).

## Common Mistakes

**BAD:** `Cache-Control: public` on personalized HTML → user A sees user B's dashboard.

**GOOD:** `private, no-store` whenever `locals.user` affects output. (Middleware can set this centrally!)

**BAD:** caching forever "CDN will fix it."

**GOOD:** every cache has an invalidation owner named in the PR.

**BAD:** `import.meta.env` cache-busted build ids that change per render.

**GOOD:** content-hash or git-SHA busting, injected at build (Module 32 integration example).

## Security Notes

- Cache poisoning: validate `Host`/forwarded headers; don't cache by user-controlled keys without bounds.
- Never cache responses containing CSRF tokens/`Set-Cookie` (mark `private, no-store`).
- Purge endpoints (webhooks) must be authenticated (Module 36).

## Performance Notes

- SWR (`swr`) serves stale while revalidating → cheap freshness.
- Memory cache is per-instance on serverless — for multi-instance, use CDN providers (🧪) or shared stores.
- Measure hit rates; a 5% hit rate is complexity without payoff.

## Exercises

**Beginner.** Static assets immutable headers verified via `curl -I` on `preview`/deploy.

**Intermediate.** `routeRules` on an SSR blog route + `Astro.cache.set` on a product route; verify `Age`/`Cache-Control` headers.

**Production.** Write-path invalidation: update action calls `cache.invalidate` by tag; integration test proves the next request is fresh.

**Architecture Challenge.** News homepage: stories change daily, "breaking" slot changes hourly, personalization for subscribers.

> **Review:** anonymous home: `maxAge: 300, swr: 3600` tag `home`; breaking slot = `server:defer` fragment cached 300s tag `breaking`; subscriber home = `no-store` variant (or fragment-based personalization on a cached shell). Invalidation on publish actions.

**Debugging Challenge.** After deploy, users still see old pages for hours. HTML cached at CDN with long TTL and no purge. Fix: short HTML TTLs + purge-on-deploy (platform hooks) or tag invalidation; hashed assets stay immutable.

## MENTAL MODEL — Caching

> Caching is **borrowed freshness** — you take latency now and promise an invalidation story later. Name all four answers (what/where/how long/who) in the design, or the cache will name them for you in an incident.

## Official Documentation

- Route caching: https://docs.astro.build/en/guides/caching/
- v7 upgrade (cache stable): https://docs.astro.build/en/guides/upgrade-to/v7/
- Sessions: https://docs.astro.build/en/guides/sessions/

## What I Should Know Before Continuing

1. The four cache questions. Answer them for your dashboard and your blog.
2. Write `routeRules` for: docs (daily), API list (10 min SWR), account (never).
3. Name the five revalidation patterns; when does each win?
4. Why is `public` caching on personalized HTML a security incident?
