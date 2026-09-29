# Module 12 — Content Modeling

> Phase 9–10 capstone of the content arc. Content is **structured data**: model it like a database schema, because that's what it is.

## Concept: modeling before markup

Every content project answers three questions first:

1. **What are my entity types?** (Post, Author, Category, Product, Doc, Changelog)
2. **What are the relations?** (Post→Author, Post→[]Tag, Product→Category)
3. **What are the URL consequences?** (each entity's public routes)

## The canonical blog model

```ts
// src/content.config.ts (excerpt — full patterns in Module 10)
const blog = defineCollection({
  loader: glob({ pattern: '**/*.{md,mdx}', base: './src/content/blog' }),
  schema: z.object({
    title: z.string().min(1),
    description: z.string().min(10).max(160),      // SERP discipline, enforced
    pubDate: z.coerce.date(),
    updatedDate: z.coerce.date().optional(),
    author: reference('authors'),
    category: reference('categories'),
    tags: z.array(z.string()).max(8).default([]),
    heroImage: image().optional(),                  // astro:content image helper
    draft: z.boolean().default(false),
    featured: z.boolean().default(false),
  }),
});
```

**Field-by-field rationale:**

| Field | Why it exists |
|---|---|
| `description` length caps | SEO snippet quality is a *content schema* concern |
| `pubDate` as `coerce.date()` | YAML dates → typed `Date`, sortable |
| `reference('authors')` | no duplicated author bios; typed relation |
| `tags` max | taxonomies rot; cap them |
| `draft` | editorial workflow — filter in **every** query *and* `getStaticPaths` |
| `heroImage: image()` | build-time optimized asset, dimensions known |

## Relations done right

```ts
const categories = defineCollection({
  loader: glob({ pattern: '**/*.json', base: './src/content/categories' }),
  schema: z.object({ title: z.string(), description: z.string() }),
});
```

- **Reference fields** (`reference('authors')`) validate existence at build.
- Reverse lookups (`authors/:id` page listing posts) = `getCollection('blog', ({data}) => data.author.id === id)`.
- For tag pages: derive the tag list once (`Set`) in `getStaticPaths` — don't create fake "tag" entries unless tags need their own editorial fields.

## Modeling product content

```ts
const products = defineCollection({
  loader: file('src/data/products.json'),
  schema: z.object({
    sku: z.string().regex(/^[A-Z0-9-]+$/),
    name: z.string(),
    priceCents: z.number().int().positive(),     // money as integer cents
    currency: z.enum(['USD', 'EUR']),
    category: reference('categories'),
    images: z.array(image()).min(1),
    attributes: z.record(z.string(), z.string()).default({}),
  }),
});
```

Notes: **money as integers**, enums for closed sets, `image()` for assets, records for open attribute bags.

## TypeScript + runtime validation: both, always

| Layer | Tool | Catches |
|---|---|---|
| Compile time | TypeScript `interface Props`, inferred `CollectionEntry` types | wiring mistakes while coding |
| Runtime | Zod schema on collections, `Astro.params`, form data, API bodies | malformed data from the world |

TypeScript types in components come **from** the schemas (`z.infer` / `CollectionEntry<'blog'>`), so there's one source of truth:

```ts
import type { CollectionEntry } from 'astro:content';
type Post = CollectionEntry<'blog'>;
```

## Organization conventions

```text
src/content/
  blog/2026-09-my-post.md        # date prefixes optional; id = path-derived
  blog/_draft-idea.md            # glob pattern '**/[^_]*.md' excludes _files
  authors/ada.json
  categories/engineering.json
```

- Prefer **files over folders-of-folders** until scale demands it.
- `_`-prefix + glob exclusion beats a `draft` flag for never-shipping drafts.
- Derived content (tag counts, related posts) computed in one shared `lib/content.ts` — not per page.

## Modeling workflow (documentation-first)

1. Write the schema **with the editorial team** (required fields = required in CMS reality).
2. Create 3 sample entries that exercise the schema (edge dates, long titles, missing optionals).
3. Generate pages; check URLs; check RSS/sitemap inputs.
4. Only then design components.

## Common Mistakes

**BAD:** `title: z.string()` as the only rule, description free-text unbounded.

**GOOD:** constraints that match editorial & SEO requirements.

**BAD:** storing rendered HTML in frontmatter (`bioHtml`) and `set:html`-ing it.

**GOOD:** Markdown fields rendered through the pipeline, or sanitized subsets.

**BAD:** separate "tags" collection with one field each for 400 tags.

**GOOD:** tags as strings + derived indexes; promote to entries only when tags get fields (descriptions, images).

**BAD:** category strings duplicated in every post (`category: "eng"`).

**GOOD:** `reference('categories')` — rename once, rename everywhere.

## Security Notes

- URLs in schemas: `z.url()`; sanitize any rich text; `image()` validates assets.
- Draft content: enforce access at the query layer **and** the route layer (SSR preview routes need auth — Module 21).
- External image URLs: restrict domains (`image.domains` / `remotePatterns`) to block SSRF-ish abuse patterns via CMS fields (Module 25/23).

## Performance Notes

- Schema validation runs at build — near-zero runtime cost, huge reliability win.
- Over-normalized content (many tiny collections) costs join-like queries in `getStaticPaths`; denormalize read-heavy fields (e.g. `authorName` snapshot) if build time suffers.

## Exercises

**Beginner.** Model `events`: `name`, `date`, `city`, `online: boolean`, `url: z.url().optional()`. List sorted; invalid file must **fail the build** with a clear Zod error (create one on purpose).

**Intermediate.** Blog + authors + categories with references; author page lists their posts; category page lists posts; both generated via `getStaticPaths`.

**Production.** Add `updatedDate` and build a `/changelog`-style "recently updated" page + `lastmod` hints for the sitemap (feed it Module 26/27 work).

**Architecture Challenge.** Newsroom: articles, live blogs (updating minute-by-minute), topics, paywalled content. Which parts are build-time collections, which live, which belong to the DB/SSR side? Compare with: *articles/topics → build-time; live blogs → live collection or SSR+cache; paywall state → server/DB only (never a content field deciding access alone).*

**Debugging Challenge.** Build fails: `ContentSchemaContainsSlugError`. A legacy schema declares `slug: z.string()`. Fix: delete the field; use `id` (or `generateId` in the loader if custom slugs are truly required).

## MENTAL MODEL — Content Modeling

> Schemas are contracts with your editors and your routes. Every field either serves a **page**, a **feed**, or a **workflow** — if it serves none, delete it. Validation at the boundary replaces debugging at the worst possible time (production, 2 AM).

## Official Documentation

- Content collections: https://docs.astro.build/en/guides/content-collections/
- Image helper in schemas: https://docs.astro.build/en/guides/images/#images-in-content-collections
- Zod (bundled): https://docs.astro.build/en/guides/schemas/ (astro/zod usage across docs)

## What I Should Know Before Continuing

1. Design a Post→Author→Category model with references. What validates relation existence, and when?
2. Why store money as integer cents? Why cap `description` at 160?
3. When do tags become a collection?
4. Where do draft checks belong so leaked drafts are impossible?
