# Module 35 — Search & Internationalization

> Phases 58–59. Two "scale" features: search (five architectures compared) and i18n (Astro's routing config + SEO).

## Part 1 — Search

### The five approaches

| Approach | How | JS cost | Scale | Freshness | Cost |
|---|---|---|---|---|---|
| **Static index + client search** (Pagefind / Fuse / MiniSearch) | index built at build; search in-browser | index chunks (lazy) | thousands of pages | rebuild | free |
| **Server search (your DB)** | `ILIKE`/tsvector/Postgres FTS | 0 (GET form) or island | millions | live | your infra |
| **Server + island UX** | action/endpoint + instant UI | island only | millions | live | infra + a bit of JS |
| **Hosted search** (Algolia/Typesense Cloud) | external index + JS SDK | SDK cost | huge | near-live | $ |
| **Hybrid** | Pagefind for docs + DB for product catalog | mixed | mixed | mixed | mixed |

**Decision inputs:** corpus size, update cadence, tolerance for JS, budget, need for typo tolerance/facets.

### Progressive-enhancement search (the Astro pattern)

```astro
---
// GET form baseline: works with JS disabled
---
<form action="/search" method="GET">
  <label for="q">Search docs</label>
  <input id="q" type="search" name="q" />
  <button>Search</button>
</form>
<SearchIsland client:idle />   <!-- enhancement: instant results, updates ?q= -->
```

`/search?q=astro+islands` is a real page (SSR or static + client filter) — shareable, crawlable, back-button-safe (Module 24). The island just makes it instant.

### Pagefind sketch (docs sites)

```bash
npx pagefind --site dist      # post-build indexing of static HTML
```

Ship only the JS needed for the search box; index loads on first query (`client:idle` island).

## Part 2 — Internationalization (i18n)

### Routing config (verified)

```js
// astro.config.mjs
export default defineConfig({
  site: 'https://example.com',
  i18n: {
    defaultLocale: 'en',
    locales: ['en', 'fr', 'de'],
    routing: {
      prefixDefaultLocale: false,     // /about  vs /fr/about  (true → /en/about)
      redirectToDefaultLocale: true,  // / → /en/ when prefixed
      // fallbackType: 'redirect' | 'rewrite' — for fallback pages (see docs)
    },
    // domains: { fr: 'https://fr.example.com' }  — SSR only (no prerendered pages)
  },
});
```

- Folder structure mirrors locales: `src/pages/about.astro` + `src/pages/fr/about.astro`.
- `Astro.currentLocale` in components; `astro:i18n` helpers (`getRelativeLocaleUrl` etc.) for links.
- **Fallback:** localized pages missing → fallback locale content (redirect or rewrite per config).

### i18n SEO

- `hreflang` alternates per page + `x-default`.
- Canonical per locale page (never canonical everything to the English URL).
- Locale-prefixed sitemaps (or one sitemap with alternates).
- Translate **metadata** (titles/descriptions), not just body.

```astro
<link rel="alternate" hreflang="fr" href="https://example.com/fr/about/" />
<link rel="alternate" hreflang="en" href="https://example.com/about/" />
<link rel="alternate" hreflang="x-default" href="https://example.com/about/" />
```

### Content organization

```text
src/content/blog/en/*.md     src/content/blog/fr/*.md
```
or frontmatter `locale` field + filtered collections. Keep slugs translated or not — decide once (translated slugs = better UX/SEO; shared ids = simpler logic).

### RTL, formatting, language switcher

- `<html dir="rtl">` for RTL locales; test CSS logical properties.
- `Intl.DateTimeFormat` / `Intl.NumberFormat` with `Astro.currentLocale`.
- Switcher preserves the current page path across locales; flag icons alone are an a11y smell — use language names.

## Common Mistakes

**BAD:** client-only search with no URL state — unshareable results.

**GOOD:** `?q=` in the URL; results page exists server-side.

**BAD:** auto-redirecting locale by `Accept-Language` with no override/remember.

**GOOD:** detect → suggest → store preference (`lang` cookie or path); always allow manual switch.

**BAD:** `hreflang` on a site with no translations.

**GOOD:** only real alternates; validate in Search Console.

**BAD:** machine-translating docs into 8 locales with no owner.

**GOOD:** locale-ownership model; partial fallbacks are fine.

## Security Notes

- Search endpoints: validate `q`, cap length, parameterize queries (Module 23).
- Locale path segments are user input: allow-list locales (`Astro.currentLocale` from config), don't reflect raw path language into HTML.
- Hosted search keys: search-only keys in browser (never admin keys).

## Performance Notes

- Pagefind-style lazy indexes keep initial JS tiny.
- DB search: FTS indexes + pagination; avoid `ILIKE '%q%'` at scale.
- i18n multiplies build size — count pages per locale; consider on-demand for huge locales.

## Exercises

**Beginner.** Static docs search with Fuse/MiniSearch over a JSON index built at build time; `?q=` URL state.

**Intermediate.** Two-locale site: `prefixDefaultLocale`, fallback page, language switcher preserving path, hreflang + per-locale canonicals.

**Production.** Postgres FTS search endpoint + island UI with keyboard navigation + `role="status"` result counts (a11y), rate-limited.

**Architecture Challenge.** Marketplace: 5 locales, 2M products, typo-tolerant search with facets.

> **Review:** product search = hosted/DB search with facets (not Pagefind); UI copy = i18n routing + translated metadata; per-locale sitemaps; search-as-you-type island in header (`client:idle`), full results page is SSR with URL params. Budget the search SDK; consider server-rendered results for SEO.

**Debugging Challenge.** French pages 404 in production with `output: 'server'` and `prerender = true` on pages — locale folders not generated because `getStaticPaths` didn't include locale params (or the fallback rewrite is misconfigured). Fix: generate all locale routes (or drop prerender for those); verify `fallbackType` config matches expectations.

## MENTAL MODEL — Search & i18n

> Search is **state in the URL plus an index somewhere** (browser, your DB, or a host) — pick the index by scale and freshness. i18n is **routing + metadata discipline**: locales are routes, alternates are contracts, and every translated URL must exist or declare its fallback.

## Official Documentation

- i18n routing: https://docs.astro.build/en/guides/internationalization/
- astro:i18n: https://docs.astro.build/en/reference/modules/astro-i18n/
- Pagefind: https://pagefind.app/ · Postgres FTS: https://www.postgresql.org/docs/current/textsearch.html

## What I Should Know Before Continuing

1. The five search approaches — pick one for 800 docs pages and one for 2M products.
2. How does search state survive reload/share in the Astro pattern?
3. `prefixDefaultLocale`, `fallbackType`, `domains` — what does each do?
4. Which i18n SEO tags must exist on every localized page?
