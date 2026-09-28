# Module 10 — Content Layer & Collections

> Phase 9. Astro's superpower: **content as typed, validated, queryable data** — from Markdown folders to CMS APIs — through one architecture (Content Layer, stable since v5, mandatory since v6).

## Concept: the Content Layer

A **collection** is a named set of entries (files or remote records) with:
- a **loader** — where data comes from (`glob()`, `file()`, custom),
- a **schema** — Zod validation + TypeScript types (strongly recommended),
- a **data store** — the layer indexes entries at build (or fetches live at request time).

```text
files/API ──loader──► content store ──schema──► typed entries ──getCollection/getEntry──► pages
```

## The config (modern API — Astro 5+; the only one since v6)

```ts
// src/content.config.ts   ← required location (NOT src/content/config.ts ⛔)
import { defineCollection, reference, z } from 'astro:content';
import { glob, file } from 'astro/loaders';

const blog = defineCollection({
  loader: glob({ pattern: '**/*.{md,mdx}', base: './src/content/blog' }),
  schema: z.object({
    title: z.string(),
    description: z.string(),
    pubDate: z.coerce.date(),
    draft: z.boolean().default(false),
    tags: z.array(z.string()).default([]),
    author: reference('authors'),          // ← relation by reference
    heroImage: z.string().optional(),
  }),
});

const authors = defineCollection({
  loader: glob({ pattern: '**/*.json', base: './src/content/authors' }),
  schema: z.object({ name: z.string(), bio: z.string(), socials: z.record(z.string(), z.url()) }),
});

export const collections = { blog, authors };
```

**Loaders:**
| Loader | Source | Use |
|---|---|---|
| `glob({ pattern, base })` | one file per entry (md, mdx, markdoc, json, yaml, toml) | blogs, docs, authors |
| `file('path')` | one file, many entries | products.json, csv (custom `parser`) |
| custom `load({ store, logger })` | anything: CMS, DB, API | third-party/remote data |
| `defineLiveCollection` (live) | request-time sources | inventory, news feeds (below) |

Entries have **`id`** (generated from filename, URL-friendly) — **not** `slug` ⛔. A schema field named `slug` throws (`ContentSchemaContainsSlugError`) in modern Astro.

## Querying (modern API)

```astro
---
// index page
import { getCollection, getEntry, render } from 'astro:content';

const posts = (await getCollection('blog', ({ data }) => !data.draft))
  .sort((a, b) => b.data.pubDate.valueOf() - a.data.pubDate.valueOf());

// single entry + relations
const first = posts[0];
const author = await getEntry(first.data.author);   // reference resolved
const { Content, headings } = await render(first);  // entry.render() ⛔ is gone
---
```

**OLD vs MODERN query cheatsheet:**

| Old | Modern |
|---|---|
| `getEntryBySlug('blog', slug)` | `getEntry('blog', id)` |
| `getDataEntryById('blog', id)` | `getEntry('blog', id)` |
| `await entry.render()` | `await render(entry)` from `astro:content` |
| `entry.slug` | `entry.id` |
| `Astro.glob('../content/blog/*.md')` | `getCollection('blog')` |
| `import { z } from 'astro:schema'` | `import { z } from 'astro/zod'` |

## Live Content Collections (request-time data)

For data that changes **between deploys** (stock, prices, news):

```ts
// src/live.config.ts
import { defineLiveCollection } from 'astro:content';

const products = defineLiveCollection({
  loader: storeLoader({ apiKey: process.env.STORE_API_KEY!, endpoint: 'https://api.store/v1' }),
  // loader implements loadCollection / loadEntry and returns data directly
});
export const collections = { products };
```

```astro
---
export const prerender = false;
import { getLiveEntry } from 'astro:content';
const { entry: product, error, cacheHint } = await getLiveEntry('products', Astro.params.id!);
if (error || !product) return new Response(null, { status: 404 });
// cacheHint can drive route caching (Module 34)
---
<h1>{product.data.name}</h1>
```

Build-time collections = fast, indexed, SEO-perfect. Live collections = fresh, request-costly. **Choose per collection.** (Stable since v6; the API mirrors the build-time one deliberately.)

## Generating routes from content

See Module 05's `[...slug].astro` pattern: `getStaticPaths` maps entries → URLs. Pagination example in Module 26.

## Why schema validation matters

- **Build-time failures** for malformed frontmatter — broken content never ships.
- **Editorial guardrails** — required `description`, ISO dates, enum categories.
- **Types for free** — `post.data.pubDate` is a `Date`, `post.data.title` is a `string`, everywhere.
- **Security** — schemas constrain what content can assert (e.g. `z.url()` for socials; never `z.string()` for URLs used in `href`).

## Common Mistakes

**BAD:** fetching a CMS in `getStaticPaths` per page with N+1 requests.

**GOOD:** a **custom loader** fetching once into the store; pages query locally.

**BAD:** `slug` field + manual slugify logic.

**GOOD:** `id` from the loader; customize via `generateId` in `glob()` if needed.

**BAD:** using live collections for a blog that changes weekly.

**GOOD:** build-time + webhook-triggered rebuild (Module 37) or route cache tags (Module 34).

**BAD:** rendering Markdown with `set:html` from raw files.

**GOOD:** collections + `render(entry)` (remark/rehype/Sätteri pipeline, syntax highlighting included).

**BAD:** one giant `posts` collection with 14 optional fields covering blog+docs+changelog.

**GOOD:** separate collections with focused schemas (`blog`, `docs`, `changelog`) — types stay honest.

## Security Notes

- Schema is a **validation layer**, not authorization. Draft filtering in queries (`!data.draft`) must match URL access (don't forget `getStaticPaths` vs SSR fetch paths).
- Custom loaders hold API keys — they run at **build/server**, never in islands.
- `z.string()` for URLs → use `z.url()`; user-supplied image paths → validate shape before `<img src>`.

## Performance Notes

- The store is indexed: `getCollection` + filter is cheap; still precompute derived views (tag counts) once per build.
- Live collections add request latency; pair with `Astro.cache` hints.
- Big collections (10k+): fine for build; prefer `getEntry` in SSR paths over loading all.

## Exercises

**Beginner.** `books` collection (glob, JSON files), schema with `title`, `author`, `year: z.coerce.number()`, render `/books/` listing sorted by year.

**Intermediate.** Blog collection with `reference('authors')`; post page resolves author + renders content + headings TOC (`render()` returns `headings`).

**Production.** Custom loader pulling GitHub releases as a `changelog` collection (fetch in `load({ store })`), typed schema, `/changelog/` page. Cache the fetch.

**Architecture Challenge.** A product catalog: 200 products updated by merchants hourly, plus 400 marketing pages updated weekly. Which collection types? Compare with: *marketing → build-time `glob`; catalog → live collection (or SSR + DB with cache tags). Never rebuild the world for price changes.*

**Debugging Challenge.** `getEntry('blog', 'my-post')` returns `undefined`; you pass a slug-like string with the wrong case/extension. IDs are generated URL-friendly from filenames (`my-post` from `my-post.md`) — log `posts.map(p => p.id)` after `astro sync` and fix; note schema `slug` fields now throw.

## MENTAL MODEL — Content

> Content is **data with a schema and an ID**, not strings to parse at runtime. Loaders define *where* it lives; the store makes it queryable; schemas make it safe and typed; pages are then trivial. Build-time for the durable, live for the volatile.

## Official Documentation

- Content collections: https://docs.astro.build/en/guides/content-collections/
- Loaders reference: https://docs.astro.build/en/reference/content-loader-reference/
- astro:content module: https://docs.astro.build/en/reference/modules/astro-content/
- Upgrade v6 (legacy removal): https://docs.astro.build/en/guides/upgrade-to/v6/

## What I Should Know Before Continuing

1. Draw the Content Layer pipeline. What do loaders, schemas, and the store each do?
2. Modern replacements for `getEntryBySlug`, `entry.render()`, `entry.slug`.
3. Build-time vs live collections: decision criteria?
4. Why is a `slug` field in a schema an error in modern Astro?
