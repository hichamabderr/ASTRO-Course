# Module 02 — Project Setup & Toolchain Deep Dive

> Phase 2. Complements orientation 02 (which gets you installed). This module teaches **what the toolchain actually does** so you can debug it.

## Concept: What is `astro dev` vs `astro build`?

Astro sits on **Vite 8** (Rolldown) for dev server + bundling, and (since v7) a **Rust compiler** for `.astro` files.

| Command | Pipeline |
|---|---|
| `astro dev` | Vite dev server (Environment API, v6+): on-demand module transform, fast HMR; pages render per request as you'd see in SSR |
| `astro build` | Rust compile `.astro` → render all routes → static `dist/` files + (with adapter) `dist/server/` entry + client assets through Rolldown |
| `astro preview` | serves `dist/` (static mode) or the adapter's preview |

**Mental model:** dev optimizes for *feedback*, build optimizes for *correctness and size*. A bug that appears only in build (e.g. stricter Rust compiler errors) is telling you something real about your HTML.

## Project structure decisions (why each folder exists)

```text
src/
  pages/         # ROUTES. File = URL. Reserved names: index.astro, [...slug].astro, 404.astro
  layouts/       # document shells (head, landmarks). Keep layouts dumb.
  components/    # reusable Astro components (HTML/CSS)
    react/       # ONLY hydrated islands live here — the folder is a budget boundary
  content/       # Markdown/MDX/data files for collections
  content.config.ts   # collection schemas + loaders (required file name)
  live.config.ts      # live collections (optional)
  actions/       # server mutations
  middleware.ts  # request pipeline
  lib/           # framework-agnostic utilities
  server/        # server-only code: db, auth, services. NEVER imported by islands.
  styles/        # global CSS (tokens, resets)
public/          # copied verbatim — favicon, robots.txt, _headers
astro.config.mjs
```

**The `server/` rule:** anything under `src/server/` must never be imported from a `client:*` island or a browser `<script>`. Enforce with code review (Module 33 shows the `PUBLIC_`-leak patterns).

## Config anatomy (verified, Astro 7)

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';
import sitemap from '@astrojs/sitemap';

export default defineConfig({
  site: 'https://example.com',      // REQUIRED for sitemap/RSS/canonical absolute URLs
  // output: 'static' is the default. Use 'server' for auth-heavy apps.
  integrations: [react(), sitemap()],
  // v7 route caching (Module 34):
  // cache: { provider: memoryCache() },
  // routeRules: { '/blog/[...slug]': { maxAge: 300, swr: 60 } },
  markdown: {
    // shikiConfig: { theme: 'github-dark' }  — Module 11
  },
  image: {
    // domains / remotePatterns for remote images — Module 25
  },
  i18n: { /* Module 35 */ },
});
```

## TypeScript configuration

`tsconfig.json` extends `astro/tsconfigs/strict`. You get:
- `.astro` type-aware completions (props, `Astro.*`),
- auto-generated types from `astro sync` (`astro:content`, `astro:i18n`, env types),
- JSX config for your island frameworks.

Run `npx astro sync` after editing `content.config.ts` if types look stale (dev also auto-syncs).

## The CLI surface you'll actually use

| Command | Use |
|---|---|
| `astro dev` / `build` / `preview` | daily |
| `astro add <integration>` | officially correct config edits |
| `astro sync` | regenerate module types |
| `astro check` | typecheck `.astro` + TS — **run in CI** |
| `npx @astrojs/upgrade` | upgrade Astro + official integrations together |

## Code: quality gates in CI

```yaml
# .github/workflows/ci.yml (excerpt)
- run: npm ci
- run: npx astro check
- run: npm run build
- run: npx playwright test   # Module 30
```

## Common Mistakes

**BAD:** copying a 2024 `astro.config.mjs` with `output: 'hybrid'` and `experimental: { session: true }`.

**GOOD:** `astro add` + the [config reference](https://docs.astro.build/en/reference/configuration-reference/); flags you don't recognize → check the upgrade guides.

**BAD:** putting `.env` secrets in `.env` with `PUBLIC_` prefix "so I can use them in the React chart."

**GOOD:** the chart fetches from your own endpoint; the key stays server-side (Module 33).

**BAD:** `npm run dev` forever; first `astro build` happens in CI where the Rust compiler rejects your unclosed `<p>` tags (v7 is strict!).

**GOOD:** build locally at least weekly; valid HTML is now a build invariant.

## Security Notes

- `.env` files: gitignore them; provide `.env.example`. Deployment platforms inject real values.
- Dependabot/`npm audit` on CI; lockfile committed.
- `astro preview` is a **dev convenience** — not a production server. Production SSR = adapter runtime (Module 37).

## Performance Notes

- Build time scales with content volume (each Markdown file is a route). Content collections are indexed to a data store — keep `pattern`s tight in loaders.
- Dev-server latency ≠ prod performance; measure on `preview`/deployed output.

## Exercises

**Beginner.** Create a fresh project; add React via `npx astro add react`; render a page with both an Astro component and a React island. Show the built `dist/` HTML.

**Intermediate.** Add `@astrojs/sitemap`, set `site`, build, and open `dist/sitemap-index.xml`. Break `site` and observe the failure mode.

**Production.** Wire the CI workflow above on a repo, including `astro check` and a build artifact upload.

**Architecture Challenge.** Your app needs: a docs area (static), an account area (SSR), and a marketing blog (static). Which `output` mode do you choose and how do you opt pages in/out? Compare with: *Choose `output: 'server'` and `export const prerender = true` on docs/blog pages, OR `output: 'static'` with `prerender = false` on account pages. Both are valid; pick the mode matching your majority. Docs/blog majority → static default; account-heavy → server default. The second is easier to reason about as the app grows.*

**Debugging Challenge.** `npx astro add react` says "integration already present" but `client:load` does nothing in production build. You installed `@astrojs/react` but never restarted dev, and in `astro.config.mjs` you have `integrations: []`. Diagnose: the CLI edit didn't land. Fix config, rebuild, verify island JS in `dist/_astro/`.

## MENTAL MODEL — Toolchain

> Dev server = fast mirrors. Build = the truth (Rust compiler, real bundles, real sizes). `src/pages/` is routing; `astro.config.mjs` is wiring; CI runs `astro check` + `build` + tests. If it builds locally, it exists; if it only works in dev, it doesn't.

## Official Documentation

- Install & setup: https://docs.astro.build/en/install-and-setup/
- CLI: https://docs.astro.build/en/reference/cli-reference/
- Config: https://docs.astro.build/en/reference/configuration-reference/
- Integrations: https://docs.astro.build/en/guides/integrations-guide/
- Directory structure: https://astro.build/basics/project-structure/ (also `docs.astro.build` tutorial section)

## What I Should Know Before Continuing

1. What differs between `astro dev` and `astro build` output? Why can v7 builds fail on HTML that dev tolerated?
2. Why does `src/server/` exist as a convention? What is the import rule?
3. Which CI steps are non-negotiable for an Astro repo?
4. Where do you look when `astro add` and a blog post disagree?
