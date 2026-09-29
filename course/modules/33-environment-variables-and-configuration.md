# Module 33 — Environment Variables & Configuration

> Phase 65. Where config comes from, what ships to the browser, and the dangerous mistakes — with Astro 6+'s **build-time inlining** rule front and center.

## The rule (Astro 6+, verified)

> `import.meta.env.*` values are **always inlined at build time**. For **runtime** server configuration, read `process.env`.

| Variable | Read via | Reaches browser? | When resolved |
|---|---|---|---|
| `PUBLIC_*` | `import.meta.env.PUBLIC_X` | ✅ yes (by design) | build |
| non-`PUBLIC` via `import.meta.env` | `import.meta.env.SECRET` ⛔ | **dangerous — inlined where imported** | build |
| Runtime secret | `process.env.SECRET` (server code only) | ❌ | **runtime** |
| `.env` files | loaded by Vite (dev/build) + your platform (runtime) | — | — |

```ts
// ✅ build-time public config
const gaId = import.meta.env.PUBLIC_GA_ID;

// ✅ runtime secret on the server
const dbUrl = process.env.DATABASE_URL!;

// ⛔ OLD pattern, now a foot-gun: SECRET gets baked into output at build
// const apiKey = import.meta.env.API_KEY;
```

**Dangerous mistakes tour:**
1. `PUBLIC_` prefix "so the island can use it" → the key is now in every visitor's JS. Islands call your endpoints instead.
2. `import.meta.env.STRIPE_SECRET` in server code on a **newer** Astro → inlined into build artifacts (it used to be a runtime read in old tutorials).
3. Committing `.env` → git history compromise; rotation hell.
4. Different value at runtime vs build → silent misconfig (feature flags). Prefer runtime config for flags that must change without rebuild.
5. Printing `process.env` wholesale into logs/error pages.

## Configuration map (who reads what, when)

```text
astro.config.mjs     build-time only (integrations, adapters, i18n, routeRules…) — never secrets for runtime
.env                 dev/build convenience; production = platform secret store
process.env          server runtime (DB URLs, API keys, SMTP…)
PUBLIC_*             intentional public constants (site name, public keys)
```

## TypeScript typing for env

```ts
// src/env.d.ts
interface ImportMetaEnv {
  readonly PUBLIC_SITE_NAME: string;
}
interface ImportMetaEnv { readonly [key: string]: string | boolean | undefined }
```

## Deployment configuration patterns

- **12-factor:** config via environment; code is portable across Node/Vercel/Netlify/Cloudflare (their secret UIs map to env vars).
- **Per-environment files** (`.env.production`) for *public, non-secret* defaults only.
- **Validation at boot:** fail fast when `DATABASE_URL` missing — a schema over `process.env` (Zod) in server entry/service init.

```ts
const Env = z.object({ DATABASE_URL: z.url(), SESSION_SECRET: z.string().min(32) });
const env = Env.parse(process.env);   // throws with a clear message at startup
```

## Common Mistakes

**BAD:** reading `import.meta.env.DATABASE_URL` "because the docs example used import.meta.env" (pre-v6 habit).

**GOOD:** `process.env` for runtime server config on Astro 6/7.

**BAD:** JWT signing key in `PUBLIC_` (yes, people do this).

**GOOD:** signing happens server-side only.

**BAD:** config scattered — `const STRIPE = 'sk_live_…'` in three files.

**GOOD:** one `src/server/config.ts` validating env once.

## Security Notes

- Anything `PUBLIC_` is **published** — treat as public in threat models.
- Secret scanning (gitleaks/trufflehog) in CI.
- Rotate on leak; keep prod/dev secrets separate; least privilege per service.

## Performance Notes

- Inlined env is free at runtime; `process.env` reads are cheap — don't cache-misuse them.
- Feature flags via env require rebuild/redeploy on Astro static pages — plan for that.

## Exercises

**Beginner.** `PUBLIC_SITE_NAME` in a page + a `SECRET` in server code; grep `dist/` to prove only the public one ships.

**Intermediate.** Zod-validated env module; unit test failure messages when vars missing.

**Production.** Platform secret rotation runbook (who, how, rollback) + gitleaks CI check.

**Architecture Challenge.** White-label SaaS needs per-tenant theme config that changes hourly. Env vars?

> **Review:** env is for *deployment* config, not *tenant data*. Tenant themes live in DB/config service, cached (Module 34); env holds connection + secrets only.

**Debugging Challenge.** Local dev shows live API data; deployed site shows 401s from the API. The key was in `.env` read via `import.meta.env` — baked at CI build time with an empty/stale value, or missing in the runtime env entirely. Fix: `process.env` + set the secret in the host's runtime environment.

## MENTAL MODEL — Configuration

> Config has three homes: **public constants** (inlined), **runtime secrets** (`process.env`, server-only), and **structural config** (`astro.config.mjs`, build-time). Confuse them and you either leak credentials or ship stale behavior. Validate env at boot; scan for leaks in CI.

## Official Documentation

- Environment variables: https://docs.astro.build/en/guides/environment-variables/
- v6 change note: https://docs.astro.build/en/guides/upgrade-to/v6/
- Configuration reference: https://docs.astro.build/en/reference/configuration-reference/

## What I Should Know Before Continuing

1. The three homes of config; which one `import.meta.env` serves in Astro 6+?
2. Why is `PUBLIC_` a security boundary, not a naming convention?
3. Boot-time env validation — what does it buy you at 3 AM?
4. Feature flags via env on a static page: what's the catch?
