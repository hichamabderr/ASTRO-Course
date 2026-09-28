# Module 37 — Deployment & Runtimes (Static, Node, Serverless, Edge, Docker)

> Phases 29/66–70. BUILD → OUTPUT → ADAPTER → RUNTIME. Each arrow is a decision.

## The pipeline

```text
astro build
  ├── dist/client/     static assets (always)
  └── dist/server/     server entry (adapter)     [only if adapter/SSR]
        ↓
ADAPTER: @astrojs/node | vercel | netlify | cloudflare
        ↓
RUNTIME: static CDN | Node process | serverless functions | edge workers
```

## Model 1 — Static hosting

**Deploy `dist/` (client) to any CDN** (GitHub Pages, S3+CloudFront, Cloudflare Pages static, Netlify static…).

- ✅ infinite scale, trivial ops, cheapest.
- ⛔ no Actions/sessions/SSR/server endpoints at runtime — plan static-compatible UX (forms POST to third-party or wait for an SSR host).

## Model 2 — Node (`@astrojs/node`)

```js
export default defineConfig({
  output: 'server',
  adapter: node({ mode: 'standalone' }),   // or 'middleware' to mount in Express
});
```

```bash
npm run build
node ./dist/server/entry.mjs        # standalone: serves client assets + server routes
```

**SSR hosting concerns checklist:**
- **Cold starts:** keep server bundle lean; warm-up strategies on serverless.
- **DB connections:** pooled per instance (Module 19).
- **Env vars:** runtime `process.env` (Module 33).
- **Logs:** structured to stdout → platform drains (Module 31).
- **Caching:** `routeRules`/CDN (Module 34).
- **Filesystem:** serverless is ephemeral — use object storage (Module 36).

## Model 3 — Serverless/edge platforms

| Adapter | Watch for |
|---|---|
| Vercel | function regions, ISR interplay with route cache semantics |
| Netlify | function bundling config |
| Cloudflare | workerd limits: Node API subset, DB drivers (use edge-compatible drivers — Neon/Turso/HTTP APIs), adapter changed a lot in v6 — read its changelog; note Astro's 2026 Cloudflare ownership trajectory |

**Edge vs Node:**

| | Node | Edge |
|---|---|---|
| Latency | region-bound | distributed near users |
| APIs | full Node | Web-standard subset |
| DB drivers | native TCP OK | HTTP/WebSocket drivers preferred |
| Startup | ms (container) | ms (worker isolate) |
| Long jobs | OK | limited CPU time |

Per-route choice is allowed (Module 14): static/edge for content, Node for heavy app routes.

## Model 4 — Docker (production recipe)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:24-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM node:24-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build          # astro build

FROM node:24-alpine AS run
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./
RUN npm ci --omit=dev
EXPOSE 4321
HEALTHCHECK CMD wget -qO- http://localhost:4321/health || exit 1
CMD ["node", "./dist/server/entry.mjs"]
```

Notes: multi-stage keeps the image small; **runtime env** injected by orchestrator (never baked); health endpoint = an Astro endpoint returning 200 (Module 15); non-root user in real prod (`USER node`); add `public/_headers`/proxy config for caches.

## Deploy pipeline (CI/CD)

```text
PR:  astro check → unit/integration → build → e2e smoke
main: full suite → build → migrate DB → deploy → post-deploy smoke (health + login flow)
content: webhook → rebuild (static) / cache purge (SSR)
```

DB migrations run **before** new code serves traffic (expand-contract pattern for zero downtime).

## Common Mistakes

**BAD:** `astro preview` as the production server.

**GOOD:** adapter's runtime (standalone Node, platform functions).

**BAD:** baking secrets into Docker images.

**GOOD:** runtime env/secret stores.

**BAD:** one giant container running cron + web + queue.

**GOOD:** separate processes with shared code.

**BAD:** assuming edge = faster for DB-heavy pages (round trips to origin DB dominate).

**GOOD:** colocate compute with data or use edge data stores.

## Security Notes

- Docker: minimal base, non-root, no secrets in layers, image scanning.
- Platform: TLS everywhere, least-privilege service accounts (deploy ≠ DB superuser).
- Ephemeral filesystem — never write uploads/exports locally.

## Performance Notes

- Static + CDN is the fastest possible model; use SSR only where the page earns it.
- Region choice near your DB matters more than near your users for SSR TTFB sometimes — measure.
- Keep function bundles small (tree-shake SDKs).

## Exercises

**Beginner.** Deploy a static build to any static host; verify headers + 404 behavior.

**Intermediate.** Node standalone in Docker (the Dockerfile above) with healthcheck; docker run locally; hit `/health`.

**Production.** Full CI/CD with migrations + post-deploy smoke tests; rollback runbook.

**Architecture Challenge.** SaaS: global content, US-East DB, EU customers (data residency). Deployment architecture?

> **Review:** content on global CDN (static); app/SSR in US-East (data locality) with edge caching for anonymous; EU tenant data in EU region schema/DB (Module 22 isolation) — or EU deployment; document the latency/legal tradeoff explicitly.

**Debugging Challenge.** 404 on `/app/dashboard` after deploying static output to a host: you expected SSR. `dist/` has no server entry — adapter missing or `output` wrong. Fix: adapter + host that runs the server entry; verify `dist/server/entry.mjs` exists in CI artifacts.

## MENTAL MODEL — Deployment

> Deployment is the **rendering strategy made physical**: static goes to CDNs, SSR goes to runtimes, edge goes near users but not always near data. Write the pipeline (check → build → migrate → deploy → smoke) once, and make every route's runtime choice match its Module 14 table.

## Official Documentation

- Deploy guide: https://docs.astro.build/en/guides/deploy/
- Adapters: https://docs.astro.build/en/guides/integrations-guide/#official-integrations
- Node adapter: https://docs.astro.build/en/guides/integrations-guide/node/
- Cloudflare adapter: https://docs.astro.build/en/guides/integrations-guide/cloudflare/

## What I Should Know Before Continuing

1. Draw BUILD → OUTPUT → ADAPTER → RUNTIME with each component's role.
2. What's unavailable on pure static hosting? How do you compensate?
3. Why use multi-stage Docker builds? Where do secrets come from?
4. Name three edge-runtime constraints that change your code.
