# 02 — Prerequisites, Environment & Project Setup

> Phase 2 of the course plan. By the end you have a verified toolchain and understand what `astro dev` and `astro build` actually do.

## Prerequisites (honest checklist)

| You need | Because |
|---|---|
| HTML semantics, forms, links | Modules 3, 5, 17, 28 assume fluent HTML |
| CSS layout (flex/grid), media queries | Module 6 |
| TypeScript generics-lite (types, interfaces, unions) | Props, schemas, Actions are typed |
| async/await, fetch, JSON | Server code and data fetching |
| git + terminal | everything |
| **Helpful:** React basics | Module 09 compares Astro vs React head-to-head |
| **Helpful:** HTTP verbs/status codes | Modules 15, 20 |
| **Helpful:** SQL concepts | Module 19 |

If you lack React, you can still complete Phases A–B and learn the *islands* model cleanly — arguably better.

## The Verified Toolchain (September 2026)

| Tool | Version | Notes |
|---|---|---|
| **Node.js** | **24.x LTS recommended** (minimum **22.12** for Astro 6/7) | Node 18/20 are unsupported by current Astro. Only even-numbered releases run Astro. |
| **Package manager** | pnpm 9+/10+ (or npm 10+, yarn 4+) | Course examples show npm; adapt freely |
| **Astro** | **7.3.x** current stable | `npx astro --version` to verify |
| **TypeScript** | 5.x (bundled with Astro templates) | strict mode on |
| **Editor** | VS Code + **Astro extension** (or Neovim/Zed with astro language server) | gives `.astro` IntelliSense + diagnostics |

```bash
node --version        # expect v22.12+ (ideally v24.x)
npm --version
```

## Create the project

```bash
npm create astro@latest my-course-project
# template: "Include sample files" is fine for exploration,
# but this course builds from "minimal" to keep every file justified.
cd my-course-project
npm install
npm run dev           # http://localhost:4321
```

Add the React integration now (the course's single primary island framework):

```bash
npx astro add react
# answer yes to installing @astrojs/react and react/react-dom,
# and yes to updating astro.config.mjs — the CLI does both.
npx astro add tailwind   # optional at this point; Module 06 covers it in depth
```

> **Official tooling matters:** `npx astro add react` writes the *current* correct config for your Astro version. Hand-editing config from a 2024 blog post is how deprecated APIs sneak in. Later upgrades: `npx @astrojs/upgrade` updates Astro **and** official integrations together.

## What the template gives you

```text
my-course-project/
├── public/               # static assets, copied as-is (favicon, robots.txt)
├── src/
│   ├── assets/           # processed assets (images optimized by astro:assets)
│   ├── components/       # .astro components (and later, islands)
│   ├── layouts/          # page shells
│   └── pages/            # ROUTES = files in this folder
├── astro.config.mjs      # integrations, adapter, site URL, output mode
├── package.json
└── tsconfig.json         # Astro's strict TS config
```

There is **no `index.html` entry point and no client `main.ts`**. `src/pages/` is the router; HTML is the product.

## The three commands that matter

| Command | What happens |
|---|---|
| `astro dev` | Vite-powered dev server; Astro components render per request in dev; HMR for CSS/islands |
| `astro build` | Prerenders static pages to `dist/`, compiles server routes with the adapter (if configured). Rust compiler + Vite 8/Rolldown in v7 |
| `astro preview` | Serves the build output locally (static preview; SSR preview requires the adapter runtime) |

Also: `astro check` (TypeScript + `.astro` diagnostics — run it in CI) and `astro sync` (regenerates `astro:content`, `astro:i18n` types after schema changes).

## Minimal `astro.config.mjs` (verified shape)

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import react from '@astrojs/react';

// https://astro.build/config
export default defineConfig({
  site: 'https://example.com', // absolute URLs for SEO (canonical, sitemap, RSS)
  integrations: [react()],
});
```

**OLD vs MODERN config traps:**

| Old (pre-v5/v6/v7) | Modern (Astro 7) |
|---|---|
| `output: 'hybrid'` | ⛔ removed (v5). Use `output: 'static'` (default) + `export const prerender = false` per page, or `output: 'server'` |
| `legacy: { collections: true }` | ⛔ removed (v6). Content Layer only |
| `experimental: { session: true }` | ⛔ sessions are stable; delete the flag |
| `experimental: { cache, routeRules, rustCompiler, advancedRouting, logger, queuedRendering }` | ⛔ all stable in v7; move `cache`/`routeRules`/`logger` to top level |
| `import { z } from 'astro:schema'` | `import { z } from 'astro/zod'` (Zod 4) |

## TypeScript setup you actually use

Astro's `tsconfig.json` extends `astro/tsconfigs/strict`. Two patterns recur all course:

```ts
// 1. Typed component props (Module 04)
interface Props {
  title: string;
  count?: number;
}
const { title, count = 0 } = Astro.props;

// 2. Typing locals populated by middleware (Module 18/21)
// src/env.d.ts
declare namespace App {
  interface Locals {
    user: { id: string; name: string } | null;
  }
}
```

## Environment variables, early and honest

Astro 6+ **inlines `import.meta.env.*` at build time**. For server-side **runtime** configuration (secrets, DB URLs that rotate), read `process.env` in server code.

```ts
// Build-time public config — fine:
const siteName = import.meta.env.PUBLIC_SITE_NAME;

// Runtime secret — NEVER import.meta.env (it is baked into output):
const dbUrl = process.env.DATABASE_URL;
```

`PUBLIC_`-prefixed vars are shipped to the browser. Everything else is server-only **as long as you never import it into an island**. Full treatment: Module 33.

## Sanity checklist before Module 01

- [ ] `npm run dev` serves the starter page
- [ ] `npx astro --version` reports 7.3.x
- [ ] `npx astro add react` completed and `astro.config.mjs` contains `react()`
- [ ] `npm run build && npm run preview` works
- [ ] Astro VS Code extension installed (`.astro` files show no red squiggles)

## MENTAL MODEL — Toolchain

> `src/pages/` is a router. `astro build` is a static-site compiler that can also emit a server. `astro.config.mjs` wires integrations (React, Tailwind, adapters). The CLI (`astro add`, `astro sync`, `astro check`, `@astrojs/upgrade`) is the officially maintained path for config changes — prefer it over copying blog posts.

## Official Documentation

- Installation: https://docs.astro.build/en/install-and-setup/
- CLI reference: https://docs.astro.build/en/reference/cli-reference/
- Configuration: https://docs.astro.build/en/reference/configuration-reference/
- Upgrade tooling: https://docs.astro.build/en/upgrade-astro/
- TypeScript: https://docs.astro.build/en/guides/typescript/

## What I Should Know Before Continuing

1. What Node versions does Astro 7 support? What happens on Node 20?
2. What replaces `output: 'hybrid'`, and how do you opt a single page into SSR on a static site?
3. Which env-var prefix reaches the browser? Where do runtime secrets come from in Astro 6+?
4. What do `astro sync` and `astro check` do, and when should CI run them?
