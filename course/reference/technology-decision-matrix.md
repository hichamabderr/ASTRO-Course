# Reference — Technology Decision Matrix (§99)

> Keep this next to the keyboard. Need → recommended Astro approach.

| Need | Recommended approach | Module |
|---|---|---|
| Static content page | Astro component + collection | 03, 10 |
| Blog / docs / changelog | Content Layer (`glob` loader, schema, `render`) | 10–12 |
| Interactive widget | Island + laziest viable `client:*` directive | 07–08 |
| Tiny interaction (toggle, menu) | Platform HTML/CSS or bundled `<script>` | 01, 06 |
| Server mutation from your UI | **Astro Action** (`defineAction`, Zod) | 16 |
| External HTTP integration / public API | **Server endpoint** | 15 |
| Webhook inbound | Endpoint + signature verify + idempotency | 36 |
| Form | Native `<form>` → Action; enhance last | 17 |
| Authentication | Session cookies + Better Auth (or Clerk) + middleware `locals` | 21 |
| Authorization | Capability checks + tenant-scoped queries in services | 22 |
| Per-user data on a page | SSR route (`prerender = false`) or `server:defer` fragment | 14, 07 |
| Fresh volatile data (prices, feeds) | Live Content Collections or SSR + short cache | 10, 34 |
| Shared client state (few islands) | URL → DOM → shared vanilla module → nanostores | 24 |
| Heavy client app zone (dashboard) | React app zone; TanStack Query inside it | 24, 09 |
| Client-heavy application overall | Consider Next.js / TanStack Start instead | 39–40 |
| Search (small corpus) | Build-time index (Pagefind/Fuse) + `?q=` URL | 35 |
| Search (large corpus) | DB FTS / hosted + island UI | 35 |
| Images | `astro:assets` `<Image>`/`<Picture>`; uploads → object storage | 25, 36 |
| SEO metadata | `SEO.astro` + canonical + JSON-LD | 26 |
| Sitemap / RSS | `@astrojs/sitemap`, `@astrojs/rss` | 26 |
| Page transition polish | `<ClientRouter />` + persist + reduced motion | 27 |
| CSS styling | Tokens + scoped CSS; Tailwind 4 via `@tailwindcss/vite` | 06 |
| Component kit (React) | Inside islands only (shadcn etc.) | 06, 09 |
| DB access | Drizzle + Postgres; services/repos; pooled client | 19 |
| Caching | Static CDN + `routeRules`/`Astro.cache` + tag invalidation | 34 |
| i18n | `i18n` config + folder-per-locale + hreflang | 35 |
| Email | Provider SDK in `src/server`, queued | 36 |
| File upload | Action + `z.instanceof(File)` + re-encode + signed URLs | 36 |
| Rate limiting | Middleware/proxy counters on mutations & auth | 23 |
| Logging/metrics | requestId middleware + structured logs + web-vitals beacon | 31 |
| Testing | Vitest pyramid + Playwright + axe | 30 |
| Deployment: content site | Static to CDN | 37 |
| Deployment: SaaS | `@astrojs/node` standalone (Docker) or Vercel/Netlify/Cloudflare adapter | 37 |
| Org-wide tooling | Custom integration (hooks) | 32 |
| Config/secrets | `PUBLIC_*` public · `process.env` runtime · validated at boot | 33 |

## The one-line rules

1. **HTML first, CSS second, Astro third, JS when necessary, framework only when necessary.**
2. Static is the default; every SSR route must pass Module 14's five questions.
3. Every island has a directive *reason* and a byte budget.
4. Every mutation is an Action (UI) or endpoint (externals) with Zod + authz inside.
5. Every cache names: what / where / how long / who invalidates.
6. Secrets never cross the server/browser boundary — enforce with folders + CI scans.
7. Verify APIs against official docs before shipping anything from memory.
