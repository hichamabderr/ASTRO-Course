# 03 — CURRENT VERIFIED ASTRO STACK (September 2026)

> Every line below was verified against **official** sources: `docs.astro.build`, the v6/v7 upgrade guides, and `withastro/astro` changelogs — at course creation time. Versions drift; treat this as a **verified snapshot** and re-verify with the method in [04](04-ecosystem-map-and-astro-evolution.md).

## Core

| Layer | Choice | Version / status | Why this choice |
|---|---|---|---|
| Framework | **Astro** | **7.3.x** ✅ (v7.0: 2026-06-22) | Content-first, islands, static+SSR in one model |
| Runtime | **Node.js** | **24.x LTS** recommended; **22.12+** minimum ✅ | Astro 6 dropped Node 18/20; only even releases |
| Language | **TypeScript** | 5.x, `astro/tsconfigs/strict` ✅ | Strict types across components, content, Actions |
| Build tooling | **Vite 8** (Rolldown) + Environment API ✅ | bundled | v7 bundles Vite 8; v6 rebuilt dev server on Environment API |
| Astro compiler | **Rust compiler** ✅ | default & only in v7 | Faster builds; **stricter HTML** (unclosed tags now error) |
| Markdown pipeline | **Sätteri** (native) ✅ default | — | remark/rehype via `@astrojs/markdown-remark` is opt-in ⚠️ legacy-leaning |
| Validation | **Zod 4** via `astro/zod` ✅ | — | `astro:schema` import ⛔ deprecated → `astro/zod` |

## UI & Styling

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Primary island framework | **React 19** + `@astrojs/react` ✅ | islands only | Widest ecosystem; course standard |
| Supported alternatives | `@astrojs/vue`, `@astrojs/svelte`, `@astrojs/solid-js`, `@astrojs/preact` ✅ | | Mixing frameworks is a **deliberate** decision (Module 09) |
| Styling | **Tailwind CSS 4** via `@tailwindcss/vite` ✅ | | v4 pattern is the **Vite plugin**, not the old `@astrojs/tailwind` integration |
| CSS baseline | Astro **scoped styles**, CSS variables, nesting, container queries ✅ | | Plain CSS is often the right answer (Module 06) |
| Component kits | shadcn/ui **inside React islands only** ⚠️ | | Never wrap static Astro content in a React design system |
| Icons | `astro-icon` or plain SVG components ✅ | | SVG import as components supported since 5.7 |

## Content

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Content architecture | **Content Layer API** ✅ | `src/content.config.ts` mandatory | `defineCollection` + `glob()`/`file()`/custom loaders from `astro/loaders` |
| Querying | `getCollection()`, `getEntry()` ✅ | | `getEntryBySlug`/`getDataEntryById` ⛔ removed; entries use `id`, not `slug` ⛔ |
| Rendering entries | `render(entry)` from `astro:content` ✅ | | `entry.render()` ⛔ removed |
| Live data | **Live Content Collections** ✅ stable since v6 | `src/live.config.ts`, `defineLiveCollection`, `getLiveCollection`/`getLiveEntry` | Request-time data with schema validation + cache hints |
| MDX | `@astrojs/mdx` ✅ | | Interactive docs: components inside content |
| Markdoc | `@astrojs/markdoc` ✅ | | stricter content authoring |
| Docs theme | **Starlight** ✅ | | production docs framework built on Astro |
| Third-party loaders | CMS/commerce loaders in the integrations directory ✅ | | Content Layer treats remote sources as collections |

## Server & Full-Stack

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Mutations | **Astro Actions** ✅ (`src/actions/index.ts`) | stable since 4.15 | `defineAction`, Zod `input`, `accept: 'form' | 'json'`, `ActionError`, `isInputError` |
| HTTP APIs | **Server endpoints** ✅ (`src/pages/**/*.ts`) | | GET/POST/PUT/PATCH/DELETE → `Response` |
| Request pipeline | **Middleware** ✅ (`src/middleware.ts`, `defineMiddleware`, `sequence`) | | `context.locals` for request-scoped data |
| Sessions | **Astro Sessions** ✅ stable | `Astro.session` / `context.session` | Default drivers on Node/Cloudflare/Netlify adapters; others via `session.driver` (unstorage) |
| Server islands | `server:defer` ✅ stable | | Deferred server rendering + `slot="fallback"` |
| Auth | **Better Auth** ✅ (official docs' lead example) | | Hosted alt: **Clerk** (official SDK). **Lucia** ⛔ is now a learning resource, not a maintained library |
| ORM | **Drizzle ORM** ✅ (course primary) | | Alternative: Prisma; direct SQL fine too |
| Database | **PostgreSQL** ✅ (Neon, Supabase, self-hosted) | | SQLite via `node:sqlite` (Node ≥ 22.5) or Turso (libSQL) |
| DB layer | **`@astrojs/db` / Astro Studio** ⛔ **removed in v7** | deprecated v6.4 | v7 guidance: Drizzle, `node:sqlite`, Turso, PlanetScale, Neon |
| Email | provider SDK/SMTP **server-side only** (Resend, SES, etc.) | | Module 36 |
| File storage | object storage + signed URLs (S3/R2/Supabase Storage) | | Module 36 |

## Delivery, Caching & Quality

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Adapters | `@astrojs/node`, `@astrojs/vercel`, `@astrojs/cloudflare`, `@astrojs/netlify` ✅ | | Node: `mode: 'standalone'` or `'middleware'` |
| Route caching | `cache: { provider: memoryCache() }` + `routeRules` + `Astro.cache.set({ maxAge, swr, tags })` ✅ **new & stable in v7** | | Tag/path invalidation; 🧪 CDN providers (Netlify/Vercel/Cloudflare) experimental; third-party e.g. Appwrite provider |
| Advanced routing | `src/fetch.ts` reserved file (v7) ✅ | | `fetchFile` config to relocate/disable |
| View transitions | `<ClientRouter />` from `astro:transitions` ✅ | | Old `<ViewTransitions />` name ⛔ deprecated; `TRANSITION_*` constants ⛔ removed in v7 → use event names (`astro:after-swap` etc.) |
| i18n | `i18n` config + `astro:i18n` + `Astro.currentLocale` ✅ | | `prefixDefaultLocale`, `fallback`, `domains` (SSR only) |
| Images | `astro:assets`: `<Image>`, `<Picture>`, `getImage()`, SVG components, responsive `layout` ✅ | | Remote images via `image().refine()`/`inferSize()` or configured domains/remotePatterns |
| Fonts | Astro **Fonts API** ✅ (experimental 5.7 → default in v6) | | Or Fontsource/self-hosted woff2 — either is valid |
| SEO | `@astrojs/sitemap`, `@astrojs/rss`, JSON-LD ✅ | | `site` config required for absolute URLs |
| Testing | **Vitest**, Testing Library, **Playwright**, axe-core ✅ | | Container API for `.astro`; `getContainerRenderer` now imported from `container-renderer` subpath (v7) |
| Checks | `astro check` (`@astrojs/check`), ESLint/Prettier ✅ | | CI must run `astro check` + build |
| Observability | structured `logger` (stable v7), request IDs, Web Vitals ✅ | | Module 31 |
| Analytics | privacy-respecting (Cloudflare Web Analytics, Plausible, etc.) + consent discipline | | Module 31 |

## Status Board (stable / experimental / removed)

### ✅ CURRENTLY STABLE (safe foundations)
Content Layer · Actions · Server islands (`server:defer`) · Sessions · Middleware · Client directives · `<ClientRouter />` view transitions · i18n routing · `astro:assets` · Live Content Collections (since v6) · CSP support (since v6) · Route caching + `routeRules` (since v7) · Advanced routing `src/fetch.ts` (since v7) · structured `logger` (since v7) · Fonts API (default since v6)

### 🧪 EXPERIMENTAL (verify flags before use)
CDN cache providers for route caching · any flag listed under `experimental` in the **current** [configuration reference](https://docs.astro.build/en/reference/configuration-reference/) (the list moves fast — check, don't memorize)

### ⛔ DEPRECATED / REMOVED (recognize old tutorials)
`@astrojs/db` + `astro db/login/logout/link/init` (removed v7) · Astro Studio (shut down) · legacy content collections & `src/content/config.ts` location (removed v6) · `Astro.glob()` (removed v6) · `entry.render()` (→ `render(entry)`) · `getEntryBySlug`/`getDataEntryById` (→ `getEntry`) · `slug` in schemas (→ `id`) · `astro:schema` (→ `astro/zod`) · `output: 'hybrid'` (removed v5) · `<ViewTransitions />` name (→ `<ClientRouter />`) · `TRANSITION_*` constants & `createAnimationScope()` (removed v7) · `getContainerRenderer()` from package roots (→ `/container-renderer`) · `import.meta.env` for runtime secrets (inlined at build since v6) · NodeApp adapter API (→ `createApp()`)

## Versions to pin in your production project

```jsonc
// package.json (indicative majors — run `npx @astrojs/upgrade` to align)
{
  "dependencies": {
    "astro": "^7.3.0",
    "@astrojs/react": "current major matched to Astro 7",
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "zod": "^4.0.0",
    "drizzle-orm": "current",
    "better-auth": "current"
  }
}
```

Do **not** freeze this table forever. Re-verify quarterly using [04](04-ecosystem-map-and-astro-evolution.md).

## MENTAL MODEL — Stack

> The stack has one center of gravity: **HTML documents assembled on the server from typed content**. Everything else — React, Tailwind, Drizzle, Better Auth — hangs off that center as a *replaceable* choice. If a dependency disappeared tomorrow, the architecture survives. If the Content Layer or the rendering model breaks, you built it wrong.

## Official Documentation

- Upgrade & versioning: https://docs.astro.build/en/upgrade-astro/
- v6 guide: https://docs.astro.build/en/guides/upgrade-to/v6/ · v7 guide: https://docs.astro.build/en/guides/upgrade-to/v7/
- Config reference (experimental flags live here): https://docs.astro.build/en/reference/configuration-reference/
- Integrations directory: https://astro.build/integrations/

## What I Should Know Before Continuing

1. Name three APIs that exist in old tutorials but were **removed** by Astro 6/7 — and their modern replacements.
2. Why must runtime secrets use `process.env` and not `import.meta.env` in Astro 6+?
3. Which database layer did Astro 7 remove, and what are the official replacements?
4. Where do you look to check whether an `experimental` flag has stabilized?
