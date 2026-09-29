# 04 — Ecosystem Map, Important Astro Evolution & How to Verify APIs

> Requirement 88 of the course brief: *"Teach me how to verify whether an Astro API is current."* This module is that skill, plus the version history you need to read old code and old tutorials safely.

## Part 1 — How to Verify Any Astro API (the skill)

When you meet an API — in docs, a blog post, a video, or generated code — run this protocol:

1. **Find the official page.** `docs.astro.build/en/guides/...` or `/en/reference/...`. Search engines rank stale mirrors and AI-generated summaries above official docs; prefer direct navigation or `site:docs.astro.build`.
2. **Check the version banner.** Official docs annotate features with **"Added in: astro@X.Y"** and call out deprecations inline. The [Upgrade Astro](https://docs.astro.build/en/upgrade-astro/) page always states the **latest release**.
3. **Check the upgrade guides for your baseline.** [v6](https://docs.astro.build/en/guides/upgrade-to/v6/) and [v7](https://docs.astro.build/en/guides/upgrade-to/v7/) are the authoritative "what changed" lists. If a tutorial predates them, it may teach removed APIs (`Astro.glob`, legacy collections, `astro:schema`, `output: 'hybrid'`).
4. **Check the config reference** for `experimental.*` flags: https://docs.astro.build/en/reference/configuration-reference/. Flags come and go per minor; this page is the truth.
5. **Changelog for fine grain:** https://github.com/withastro/astro/blob/main/packages/astro/CHANGELOG.md. Also `withastro/adapters`, `withastro/integrations` changelogs for adapter majors.
6. **Old docs snapshots exist:** `v6.docs.astro.build` is an unmaintained snapshot — useful *only* for maintaining old projects.

**Smell test for stale tutorials (2024–2025 era):**

| You see this | It's teaching | Modern equivalent |
|---|---|---|
| `src/content/config.ts` | legacy location | `src/content.config.ts` |
| `getCollection('blog', ({ slug }) =>` / `post.slug` | legacy API | `entry.id` |
| `const { Content } = await post.render()` | legacy API | `const { Content } = await render(post)` from `astro:content` |
| `import { z } from 'astro:schema'` | pre-Zod-4 | `import { z } from 'astro/zod'` |
| `output: 'hybrid'` | pre-v5 | `output: 'static'` + `prerender = false` |
| `<ViewTransitions />` | pre-5.x naming | `<ClientRouter />` from `astro:transitions` |
| `astro db push --remote`, `astro login` | Astro Studio era | Drizzle + your database provider |
| `Astro.glob('./posts/*.md')` | pre-Content-Layer | `getCollection()` or `import.meta.glob()` |
| `import.meta.env.DATABASE_URL` in server code | pre-v6 assumption | `process.env.DATABASE_URL` |
| `experimental: { session: true }` | 5.1 era | sessions stable — delete flag |

## Part 2 — Important Astro Evolution (verified from official upgrade guides)

### Astro 3 → 4 (2023–2024)
- View Transitions (later renamed), Image optimizations matured, Actions previewed in 4.15.

### Astro 5 (Nov 2024) — **the Content Layer release**
- ✅ **Content Layer API:** `src/content.config.ts`, loaders (`glob()`, `file()`, custom), `reference()`, unified `getEntry()`, `render(entry)`, entries identified by **`id`**.
- ⛔ `output: 'hybrid'` **removed**. Two modes: `static` (default, per-page `prerender = false`) and `server` (per-page `prerender = true`).
- ✅ **Server Islands** (`server:defer`) stable.
- 🧪 Sessions (5.1) and Live Content Collections (5.10) introduced experimental; responsive images experimental.

### Astro 6 (Feb 2026) — **the platform release**
- **Vite 7 + rebuilt dev server** on Vite's **Environment API** (dev ≈ production runtime).
- **Zod 4**; `astro:schema` → **`astro/zod`**.
- **Node ≥ 22.12** only.
- ⛔ **Legacy collections fully removed**; `src/content.config.ts` mandatory. `Astro.glob()` removed. `getEntryBySlug`/`getDataEntryById` removed.
- ✅ **Live Content Collections stable.** ✅ **CSP support stable.** ✅ **Sessions stable** (drivers auto-configured on Node/Cloudflare/Netlify adapters).
- ⚠️ **`import.meta.env` values are always inlined at build time** — runtime config must use `process.env`.
- 🧪 Route caching (`cache`, `routeRules`) experimental; Rust compiler, advanced routing, queued rendering, structured logger behind flags.

### Astro 7 (June 2026) — **the infrastructure release**
- **Vite 8** (Rolldown bundler).
- **Rust compiler** is the default *and only* `.astro` compiler: **stricter HTML** (unclosed tags error; invalid nesting no longer auto-corrected).
- **Sätteri** native Markdown pipeline is the default. remark/rehype plugins require `@astrojs/markdown-remark` + `markdown: { processor: unified() }` (config `remarkPlugins` etc. deprecated).
- **Route caching stable** at top level: `cache: { provider: memoryCache() }`, `routeRules`, `Astro.cache.set({ maxAge, swr, tags })`, `context.cache`, tag/path invalidation.
- **Advanced routing stable** — `src/fetch.ts` is a reserved filename (relocate via `fetchFile` config or `null` to disable).
- `logger` stable (top-level config). `compressHTML` default is now `'jsx'` (whitespace rules changed).
- ⛔ **`@astrojs/db` removed** (with `astro db`, `astro login`, `astro logout`, `astro link`, `astro init`). Official replacements: **Drizzle ORM**, **`node:sqlite`**, Turso/Neon/PlanetScale.
- ⛔ `astro:transitions` internals removed (`TRANSITION_*` constants, `isTransition*Event()`, `createAnimationScope()`) → use lifecycle event names: `astro:before-preparation`, `astro:before-swap`, `astro:after-swap`, `astro:page-load`, …
- ⚠️ `getContainerRenderer()` deprecated at package root → `@astrojs/react/container-renderer` etc.

### Industry context (September 2026)
- **Cloudflare acquired The Astro Technology Company (Jan 2026).** Astro stays MIT open source; core team at Cloudflare. Watch the Cloudflare adapter/runtime story, but the course does not assume Cloudflare hosting — Node/Vercel/Netlify are covered equally.
- Weekly releases continue (7.3.x at course creation). **Pin majors, upgrade minors with `npx @astrojs/upgrade`, re-verify this course's stack table each quarter.**

## Part 3 — Ecosystem Map (who owns what)

```text
OFFICIAL (withastro)        astro, integrations (react/vue/svelte/solid/preact/mdx/markdoc),
                            adapters (node/vercel/cloudflare/netlify), sitemap, rss,
                            check, upgrade CLI, Starlight, Astro compiler

FIRST-PARTY ECOSYSTEM       Tailwind (v4 vite plugin), Zod (bundled as astro/zod),
                            Drizzle, Playwright, Vitest — industry standards Astro builds on

COMMUNITY                   loaders (CMS/commerce), auth helpers (Better Auth docs example),
                            cache providers (e.g. Appwrite CDN), Starlight themes,
                            nanostores/zustand, search libs (Pagefind, Fuse)

REMOVED LAYER (historical)  @astrojs/db, Astro Studio, legacy collections, astro:schema
```

**Selection rules:**
1. Official integration first. Community package second (check maintenance + Astro-major compatibility). Hand-rolled third.
2. One UI framework for islands per project unless there is a written justification.
3. A dependency must buy more than it costs: bundle, upgrade churn, and security surface count as costs.

## MENTAL MODEL — Evolution

> Astro's *philosophy* (documents, islands, zero-JS default) has been stable since 1.0. Its *APIs* churn at major versions, and the direction of travel is consistent: **content is a typed data layer**, **caching and rendering are per-route decisions**, **the compiler enforces real HTML**, and **managed platform services (Studio) are out — bring your own database**. Learn the philosophy permanently; verify the APIs continuously.

## Official Documentation

- Upgrade guides index: https://docs.astro.build/en/upgrade-astro/
- v6: https://docs.astro.build/en/guides/upgrade-to/v6/ · v7: https://docs.astro.build/en/guides/upgrade-to/v7/
- Changelog: https://github.com/withastro/astro/blob/main/packages/astro/CHANGELOG.md
- Integrations directory: https://astro.build/integrations/
- Roadmap discussions: https://github.com/withastro/roadmap/discussions

## What I Should Know Before Continuing

1. Walk the five-step protocol for verifying an API. Which page is authoritative for experimental flags?
2. What did Astro 5 change about output modes? What did Astro 6 remove from content collections? What did Astro 7 remove from the DB story?
3. Why does `import.meta.env` no longer work for runtime secrets?
4. A teammate's 2025 tutorial uses `post.render()` and `post.slug`. What do you replace them with?
