# Module 26 — SEO, Sitemaps & RSS

> Phases 21, 39–41. Astro sites are HTML-first — which is exactly what crawlers want. This module builds the complete SEO layer: metadata, structured data, sitemap, robots, RSS.

## Concept: SEO is server-rendered HTML + honest metadata

```text
Crawler → HTML (already complete: text, links, titles) → index
        → sitemap.xml (discovery) → rss.xml (syndication)
        → JSON-LD (understanding)
```

SPA-era SEO tricks (prerender services, dynamic meta injection) are unnecessary. Get the document right.

## The head component (build it once)

```astro
---
// src/components/SEO.astro
interface Props {
  title: string;
  description: string;
  canonical?: URL | string;
  image?: string;            // absolute URL, 1200x630
  type?: 'website' | 'article';
  noindex?: boolean;
  jsonLd?: Record<string, unknown> | Record<string, unknown>[];
}
const { title, description, canonical = Astro.url, image, type = 'website', noindex, jsonLd } = Astro.props;
const siteName = 'Astro Course';
const canonicalURL = new URL(canonical, Astro.site);
const imageURL = image ? new URL(image, Astro.site) : new URL('/og-default.png', Astro.site);
---

<title>{title} · {siteName}</title>
<meta name="description" content={description} />
<link rel="canonical" href={canonicalURL} />
{noindex && <meta name="robots" content="noindex,nofollow" />}

<meta property="og:type" content={type} />
<meta property="og:title" content={title} />
<meta property="og:description" content={description} />
<meta property="og:url" content={canonicalURL} />
<meta property="og:image" content={imageURL} />
<meta name="twitter:card" content="summary_large_image" />

{jsonLd && (
  <script type="application/ld+json" set:html={JSON.stringify(jsonLd)} />
)}
```

(`set:html` is *correct* here — the value is JSON you constructed, `JSON.stringify` escapes `</script>` risk contexts properly when composed carefully; never pass user HTML.)

## Structured data (JSON-LD) examples

```js
// Article
{
  '@context': 'https://schema.org', '@type': 'Article',
  headline: post.data.title, datePublished: post.data.pubDate.toISOString(),
  author: { '@type': 'Person', name: author.data.name },
  image: imageURL,
}
// BreadcrumbList, Organization, Product, FAQPage — same pattern
```

Validate with Google's Rich Results Test.

## Sitemap (`@astrojs/sitemap`)

```bash
npx astro add sitemap
```

```js
export default defineConfig({
  site: 'https://example.com',   // REQUIRED
  integrations: [sitemap({ filter: (page) => !page.includes('/admin'), })],
});
```

- Produces `sitemap-index.xml` + segment sitemaps at build.
- For huge/SSR sites: generate a sitemap from your route registry (endpoint `/sitemap.xml`) — or feed-based dynamic sitemaps.
- `robots.txt` in `public/`:

```text
User-agent: *
Allow: /
Disallow: /app/
Disallow: /admin/
Sitemap: https://example.com/sitemap-index.xml
```

## RSS (`@astrojs/rss`)

```ts
// src/pages/rss.xml.ts
import rss from '@astrojs/rss';
import { getCollection } from 'astro:content';

export async function GET(context) {
  const posts = await getCollection('blog', ({ data }) => !data.draft);
  return rss({
    title: 'Astro Course Blog',
    description: 'Content-first engineering',
    site: context.site ?? 'https://example.com',
    items: posts.map((p) => ({
      title: p.data.title,
      description: p.data.description,
      pubDate: p.data.pubDate,
      link: `/blog/${p.id}/`,
      categories: p.data.tags,
    })),
    customData: '<language>en-us</language>',
  });
}
```

## Content URLs & pagination SEO

- Descriptive, stable slugs (`/blog/astro-islands/` not `/p/123`).
- Paginated routes are real pages (`/blog/page/2/`) with self-canonicals; or `?page=` with canonical rules — pick one, be consistent.
- Related articles: real links (crawlable), not JS-only.
- Internal linking: docs/nav/footer links pass crawl depth; orphan pages don't index.

## Performance ↔ SEO

Core Web Vitals are ranking-adjacent *and* conversion drivers: fast HTML (static), optimized images (Module 25), zero-JS baseline (Module 07), fonts handled (Module 29).

## Common Mistakes

**BAD:** identical `<title>` on every page ("Home").

**GOOD:** unique title/description per route from content data.

**BAD:** canonical to `http://localhost` in prod (missing `site`).

**GOOD:** `site` in config + absolute canonicals.

**BAD:** `noindex` leaked into production layouts from staging flags.

**GOOD:** `noindex` only for app/admin/preview routes; test robots meta in CI.

**BAD:** JSON-LD with `set:html` built from raw user strings without `JSON.stringify`.

**GOOD:** stringify structured data objects.

## Security Notes

- Don't leak private URLs in sitemaps/RSS (filter like `robots.txt`).
- `set:html` for JSON-LD: always `JSON.stringify` of *your* objects.

## Performance Notes

- Sitemap/RSS are build artifacts — free at runtime.
- OG images: compress; they're fetched by scrapers constantly.

## Exercises

**Beginner.** `SEO.astro` used on 3 pages; verify with curl that titles/canonicals differ.

**Intermediate.** Blog: Article JSON-LD + sitemap + RSS + `robots.txt`; validate JSON-LD.

**Production.** Pagination `/blog/page/[n]` with correct rel-next/prev or canonical strategy + BreadcrumbList JSON-LD + OG images per post (build-time `getImage`).

**Architecture Challenge.** Docs in 3 locales + app area. Canonical & hreflang plan?

> **Review:** `hreflang` alternates per doc (`x-default` to default locale), locale-prefixed canonicals per i18n config (Module 35), app area `noindex`, sitemap per locale (or one sitemap with alternates). RSS per locale.

**Debugging Challenge.** Google shows "https://example.com/" as canonical for every page. `<link rel="canonical">` was hardcoded in the layout. Fix: canonical = `Astro.url` per page via `SEO.astro`.

## MENTAL MODEL — SEO

> SEO is **the document told the truth**: unique titles, absolute canonicals, complete HTML, discoverable URLs (sitemap/RSS), machine-readable meaning (JSON-LD). There is no trick layer — just metadata discipline at render time.

## Official Documentation

- Sitemap: https://docs.astro.build/en/guides/integrations-guide/sitemap/
- RSS: https://docs.astro.build/en/guides/rss/
- Meta tags & SEO recipes: https://docs.astro.build/en/recipes/build-custom-img-component/ (see also tutorial SEO section)
- Structured data (schema.org): https://schema.org/

## What I Should Know Before Continuing

1. Build `SEO.astro` from memory — every tag and why it exists.
2. What does `site` config affect? What breaks without it?
3. Sitemap vs robots.txt vs RSS — three discovery roles.
4. Where does `set:html` appear legitimately in SEO work?
