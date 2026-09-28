# Module 32 — Integrations & Vite (plus building a custom integration)

> Phases 28/63–64/91. What an Astro integration can do, how Astro relates to Vite, and a working custom integration.

## Concept: what is an integration?

An **integration** is a package that hooks Astro's lifecycle to extend it:

```js
function myIntegration(options) {
  return {
    name: 'my-integration',
    hooks: {
      'astro:config:setup': ({ updateConfig, injectScript, addClientDirective, logger }) => { /* … */ },
      'astro:config:done': ({ config }) => { /* … */ },
      'astro:route:setup': ({ route }) => { /* per-route hooks */ },
      'astro:build:start': () => { /* … */ },
      'astro:build:done': ({ dir, pages }) => { /* write files! */ },
      'astro:server:setup': ({ server }) => { /* dev server */ },
      'astro:server:done': () => { /* … */ },
      'astro:middleware:setup': ({ addMiddleware }) => { /* … */ },
    },
    // adapters additionally implement supportedAstroFeatures etc.
  };
}
export default myIntegration;
```

What integrations do in the wild: add framework renderers (React), CSS tooling (Tailwind v4 is via Vite plugin instead), sitemap/RSS generators, markdown plugins, route rules, custom client directives (`addClientDirective`), build artifacts, dev toolbar apps.

## Astro ↔ Vite (the actual relationship)

```text
Astro compiler (Rust): .astro → JS module + template
        ↓
Vite 8 (Rolldown): dev server (Environment API), transforms, HMR, bundling
        ↓
dist/_astro/*.js|css  + prerendered HTML
```

- `vite: { plugins: [...], resolve: { alias } }` in `astro.config.mjs` — Vite config nested under `vite`.
- Env vars: Vite's `import.meta.env` is **build-inlined in Astro 6+**; runtime config = `process.env` (Module 33).
- Server vs client behavior: anything imported into islands goes through the **client** build; frontmatter-only imports are **server/build** bundles. Vite's Environment API (v6+) is why dev ≈ prod runtime.

## A small custom integration (build it)

**Goal:** inject a `x-build-id` script and a build manifest JSON.

```ts
// integrations/build-id.ts
import type { AstroIntegration } from 'astro';

export default function buildId(): AstroIntegration {
  let buildId = '';
  return {
    name: 'build-id',
    hooks: {
      'astro:config:setup': ({ injectScript }) => {
        buildId = Date.now().toString(36);
        // 'head-inline' | 'head-hoisted' | 'body-inline' | 'before-hydration' — see docs for current kinds
        injectScript('head-inline', `window.__BUILD_ID__ = ${JSON.stringify(buildId)};`);
      },
      'astro:build:done': ({ dir }) => {
        // write dist/build-manifest.json using node:fs/promises
      },
    },
  };
}
```

```js
// astro.config.mjs
import buildId from './integrations/build-id';
export default defineConfig({ integrations: [buildId()] });
```

**Hooks worth knowing** (full list in the [Integrations API reference](https://docs.astro.build/en/reference/integrations-reference/)): `astro:config:setup` (inject scripts/styles, add renderers/directives, update config), `astro:route:setup` (per-route config), `astro:build:done` (emit files, read page list), `astro:middleware:setup`, plus adapter-only APIs (`supportedAstroFeatures`, entrypoints).

## When to build one

- You apply the same config/code to **every project** (org-level tooling).
- You're packaging something reusable for the ecosystem (a loader! — Module 10's custom loaders are the content-flavored sibling of integrations).
- Otherwise: a `lib/` function is simpler.

## Common Mistakes

**BAD:** forking Astro behavior via fragile Vite plugins that touch internal modules.

**GOOD:** integration hooks first; Vite plugins only for standard transforms.

**BAD:** integration that writes secrets into injected scripts.

**GOOD:** inject *public* config only.

**BAD:** depending on removed APIs (e.g. old `getContainerRenderer` import path).

**GOOD:** v7 paths (`@astrojs/react/container-renderer`) and `npx @astrojs/upgrade` discipline.

## Security Notes

- Integrations run with **full build privileges** (filesystem, network) — audit third-party ones like supply-chain code.
- `injectScript('head-inline')` content must be XSS-safe.

## Performance Notes

- Every integration adds build-time work; measure `astro build` before/after.
- Prefer build-time output (manifests, sitemaps) over runtime hooks where possible.

## Exercises

**Beginner.** Use `injectScript` in a custom integration to print build mode (`dev`/`prod`) to console.

**Intermediate.** Integration that adds a `x-robots-tag: noindex` header to `/preview/*` routes via `astro:route:setup`/middleware injection.

**Production.** Publish an internal integration that generates `dist/version.json` (git SHA) and injects `window.__VERSION__`; consume it in two projects.

**Architecture Challenge.** Team wants a "design-system integration" auto-importing components. Evaluate vs plain imports/aliases.

> **Review:** auto-imports obscure provenance and break LSP; prefer explicit imports + a barrel file + codemods. Integrate only *process* (linting, tokens export, build manifests), not magic.

**Debugging Challenge.** Vite alias works in dev, breaks the build of islands only. Alias applied in a plugin running only on the server environment; client bundle missing it. Fix: set alias in top-level `vite.resolve` (applies to both) and verify with a build.

## MENTAL MODEL — Integrations

> Integrations are **lifecycle plugins with admin rights** — they shape config, scripts, routes, and build output. Use them to encode org-wide policy; use libraries for app code. Vite is the muscle under Astro: when something behaves differently in dev and build, think *environments*.

## Official Documentation

- Integrations guide: https://docs.astro.build/en/guides/integrations-guide/
- Integrations API: https://docs.astro.build/en/reference/integrations-reference/
- Vite docs: https://vite.dev/ (Vite 8 migration for plugin authors)

## What I Should Know Before Continuing

1. Name five hook moments and one use for each.
2. Where do Vite plugins go in `astro.config.mjs`? Where do env vars get inlined?
3. Integration vs custom loader vs plain `lib/` — decision rule.
4. Why do integrations deserve a supply-chain audit?
