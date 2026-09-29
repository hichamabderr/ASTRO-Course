# Module 05 — Pages, Layouts & Routing

> Phase 4. `src/pages/` IS the router: static routes, dynamic `[param]`, rest `[...slug]`, 404s, redirects, rewrites, endpoints — and how routes become static files or SSR handlers.

## Concept: the file system is the route table

```text
src/pages/index.astro            →  /
src/pages/about.astro            →  /about
src/pages/blog/index.astro       →  /blog/
src/pages/blog/[...slug].astro   →  /blog/anything/nested
src/pages/products/[id].astro    →  /products/123
src/pages/404.astro              →  404 page
src/pages/api/posts.ts           →  GET/POST /api/posts (endpoint, Module 15)
```

- **Static routes** — one file → one page.
- **Dynamic routes** — `[param]` requires `getStaticPaths()` when prerendered (to enumerate pages) or `Astro.params` at request time in SSR.
- **Rest params** — `[...slug]` matches zero-or-more segments (`/blog`, `/blog/a`, `/blog/a/b`).
- Trailing slash behavior is config-driven (`trailingSlash: 'ignore' | 'always' | 'never'`) and matters for canonical URLs (Module 26).

## Dynamic routes in both worlds

### Prerendered (static) — enumerate at build

```astro
---
// src/pages/blog/[...slug].astro
import { getCollection, render } from 'astro:content';
import BaseLayout from '../../layouts/BaseLayout.astro';

export async function getStaticPaths() {
  const posts = await getCollection('blog', ({ data }) => !data.draft);
  return posts.map((post) => ({
    params: { slug: post.id },     // Content Layer: id, not slug ⛔
    props: { post },
  }));
}

const { post } = Astro.props;
const { Content } = await render(post);
---

<BaseLayout title={post.data.title}>
  <article><Content /></article>
</BaseLayout>
```

### SSR (on-demand) — read the request

```astro
---
// same file with: export const prerender = false;
const { id } = Astro.params;               // string | undefined
const product = await db.product.find(id); // Module 19
if (!product) return new Response(null, { status: 404 });
---
<h1>{product.name}</h1>
```

**Optional catch-all:** `src/pages/[[...path]].astro` matches `/` too (verify support in the routing reference for your version — the pattern has been supported since 4.x).

## Redirects & rewrites

```js
// astro.config.mjs
export default defineConfig({
  redirects: {
    '/old-blog/[...slug]': '/blog/[...slug]',
    '/promo': { status: 302, headers: { 'x-robots-tag': 'noindex' } },
  },
});
```

- **Redirects** (`redirects` config) generate real HTTP redirects (or meta-refresh fallback on static hosts).
- **Rewrites** (`Astro.rewrite()` in SSR) serve *another route's* rendering for the current URL — i18n fallbacks, A/B paths, legacy URL support. (Advanced routing in v7 adds `src/fetch.ts` configuration — see the [routing guide](https://docs.astro.build/en/guides/routing/) for current capabilities; don't guess APIs beyond what docs show.)

## Layouts: the document shell

```astro
---
// src/layouts/BaseLayout.astro
import '../styles/global.css';
interface Props { title: string; description?: string }
const { title, description = '' } = Astro.props;
const canonical = new URL(Astro.url.pathname, Astro.site);
---

<!doctype html>
<html lang={Astro.currentLocale ?? 'en'}>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>{title}</title>
    <meta name="description" content={description} />
    <link rel="canonical" href={canonical} />
    <slot name="head" />
  </head>
  <body>
    <a class="skip-link" href="#main">Skip to content</a>
    <header>…nav…</header>
    <main id="main"><slot /></main>
    <footer>…</footer>
  </body>
</html>
```

Layouts own: landmarks, skip links, global CSS import, fonts, ClientRouter decision (Module 27), SEO defaults (Module 26).

## 404 & error UX

`src/pages/404.astro` is statically generated in `static` output (served by hosts with custom 404 config). In SSR, middleware can rewrite 404/500 (see `@astrojs/node` changelog pattern: fetch `dist/404.html` on 404 status in standalone mode).

## Common Mistakes

**BAD:** `getStaticPaths` on an SSR route "to be safe."

**GOOD:** `getStaticPaths` is for prerendered dynamic routes only; SSR reads `Astro.params`.

**BAD:** generating `/blog/my-post` and `/blog/my-post/` inconsistently; canonical points at the wrong one.

**GOOD:** pick `trailingSlash` policy, enforce it, and make canonical absolute & consistent (Module 26).

**BAD:** client-side routing (`history.pushState`) to "make it an app."

**GOOD:** real links + optional `<ClientRouter />` enhancement (Module 27). Real URLs are shareable, crawlable, and cacheable.

**BAD:** a `[...slug].astro` with hand-rolled slug parsing across 5 collections.

**GOOD:** explicit routes per content type (`/blog/[id]`, `/docs/[...slug]`) or a careful single catch-all with typed dispatch.

## Security Notes

- `Astro.params`, `Astro.url.searchParams`, and rewrite targets are **user input**. Validate with Zod; never interpolate into `set:html`, SQL (parameterize! Module 19), or shell.
- Open redirects: don't build `redirect(Astro.url.searchParams.get('next'))` without allow-listing destinations (Module 23).
- `404.astro` must not leak internals (stack traces, hostnames).

## Performance Notes

- Route count = build cost. Tens of thousands of prerendered pages → watch build time; consider on-demand for huge catalogs (Module 13).
- Each route bundles shared CSS/JS once (`_astro/` chunks) — layouts help dedupe.

## Exercises

**Beginner.** Pages `/`, `/about`, `/pricing` with a shared layout. Add `404.astro` and visit a bogus URL in `preview`.

**Intermediate.** `src/pages/team/[member].astro` prerendered from a JSON array via `getStaticPaths`, passing member data as props.

**Production.** Blog routes from a content collection (`[...slug]`), plus `redirects` from `/articles/*` and a canonical tag per page.

**Architecture Challenge.** 50k product pages, changing prices hourly. All static? Compare with: *static shells are out — SSR or Live Collections for price, but consider `routeRules` caching (Module 34) + prerendered marketing/product docs. Rendering strategy is per-route, and price is a runtime fact (Module 14).*

**Debugging Challenge.** `/blog/hello-world` 404s in prod but works in dev. `getStaticPaths` returns `params: { slug: 'hello-world' }` while the file is `[...slug].astro` (expects `slug: 'hello-world'` **or** array) — actually works — but the real bug: you passed `params: { slug: post.data.slug }` while the collection defines `id`-based URLs. Align params with `post.id` and rebuild.

## MENTAL MODEL — Routing

> A file in `src/pages/` is a **contract with a URL**. Static routes are promises kept at build time; dynamic routes are promises kept by `getStaticPaths` (build) or `Astro.params` (request). Redirects and rewrites are contracts too. Everything a crawler or user can type maps to a file — that's the whole system.

## Official Documentation

- Pages: https://docs.astro.build/en/basics/pages/
- Routing: https://docs.astro.build/en/guides/routing/
- Redirects: https://docs.astro.build/en/guides/routing/#redirects
- API reference (Astro global): https://docs.astro.build/en/reference/api-reference/
- i18n routing: https://docs.astro.build/en/guides/internationalization/

## What I Should Know Before Continuing

1. When is `getStaticPaths` required? When is `Astro.params` enough?
2. `[param]` vs `[...param]` — what does each match?
3. Redirect vs rewrite: who sees what?
4. Why must URL params be validated even though routes are "your own files"?
