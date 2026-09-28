# Module 18 — Middleware

> Phase 15. `src/middleware.ts` runs **before (and after)** every request — the place to build request context: sessions, user, tenant, logs, headers.

## Concept & anatomy

```ts
// src/middleware.ts
import { defineMiddleware, sequence } from 'astro:middleware';

const auth = defineMiddleware(async (context, next) => {
  // BEFORE: runs for pages, endpoints, actions
  const session = await context.session?.get('userId');
  context.locals.user = session ? await findUser(session) : null;

  const response = await next();   // render the route (or next middleware)

  // AFTER: inspect/modify the response
  response.headers.set('x-request-id', context.locals.requestId);
  return response;
});

const logging = defineMiddleware(async (context, next) => {
  const start = performance.now();
  const response = await next();
  console.log(JSON.stringify({
    level: 'info', requestId: context.locals.requestId,
    path: context.url.pathname, status: response.status,
    ms: Math.round(performance.now() - start),
  }));
  return response;
});

export const onRequest = sequence(logging, auth);
```

**Verified API surface:** `defineMiddleware(fn)`, `sequence(...middlewares)`, `onRequest` export, `context.locals` (persisted into pages/actions/endpoints as `Astro.locals`/`ctx.locals`), `next()` (can take a **rewrite** argument — see docs), `context.rewrite()` for serving another route's render.

## Type the locals

```ts
// src/env.d.ts
declare namespace App {
  interface Locals {
    user: { id: string; role: 'user' | 'admin' } | null;
    tenant: { id: string; plan: string } | null;
    requestId: string;
  }
}
```

Now every page, endpoint, and action sees typed `locals` — this is the backbone of Modules 21–22.

## The canonical pipeline

```text
Request
  ↓ logging (request ID, timing)
  ↓ security headers / CSP
  ↓ session parse → locals.user / locals.tenant
  ↓ (route guards live in pages/actions — NOT only here)
  ↓ next()
Response ← header tweaks, cache directives
```

## Pattern: establish `locals.user` / `locals.session` / `locals.tenant`

```ts
export const onRequest = sequence(security, logging, sessionContext);

const sessionContext = defineMiddleware(async (ctx, next) => {
  const sid = ctx.cookies.get('session')?.value;
  if (sid) {
    const session = await sessions.verify(sid);       // server-side (Module 21)
    ctx.locals.user = session?.user ?? null;
    ctx.locals.tenant = session?.tenant ?? null;
  } else {
    ctx.locals.user = null;
    ctx.locals.tenant = null;
  }
  return next();
});
```

## What middleware is NOT

- **Not the authorization layer.** A guard here (`if (!user) return redirect()`) protects *pages that forget nothing* — but Actions/endpoints called directly must re-check. Defense in depth: middleware redirects UX; handlers enforce.
- **Not a place for response body parsing** (expensive; you usually don't need it).
- **Not per-route** — it runs for everything; keep it fast (no heavy queries on public paths; lazy-load user details only when needed).

## Rewrites & response interception

```ts
const i18nFallback = defineMiddleware(async (ctx, next) => {
  if (ctx.url.pathname.startsWith('/docs/')) {
    // e.g. serve default-locale docs for missing translations
    // ctx.rewrite('/en' + ctx.url.pathname) — see current docs for the rewrite API
  }
  return next();
});
```

`next()` / `ctx.rewrite()` let one URL render another route's output — i18n fallbacks, legacy URLs, A/B layouts. (Verify current signature in the [middleware guide](https://docs.astro.build/en/guides/middleware/); v7 advanced routing adds `src/fetch.ts` configuration.)

## Common Mistakes

**BAD:** `if (!locals.user) return redirect('/login')` middleware covering `/api/*` and breaking webhooks.

**GOOD:** explicit route matching (`startsWith('/app')`) or per-route guards.

**BAD:** DB query for user on every asset-ish request.

**GOOD:** only when cookie present; cache session lookups (Module 34).

**BAD:** swallowing errors: `catch {}` around `next()` returning blank 200s.

**GOOD:** log with request ID, return 500 page/JSON, never leak stack traces.

**BAD:** trusting `x-user-id` header from the client "for microservices."

**GOOD:** only trust headers set by *your* infrastructure (strip inbound spoofable headers first).

## Security Notes

- Middleware is the right place for: security headers (CSP, HSTS, X-Frame-Options), origin checks (CSRF), rate limiting counters, stripping spoofed headers.
- `locals` is server-only — safe for user objects; never `define:vars` them into scripts.
- Redirect targets: never redirect to unvalidated `?next=` values (open redirect — Module 23).

## Performance Notes

- Middleware latency is on **every** request including SSR pages — keep it allocation-light.
- Set `Cache-Control` for anonymous vs logged-in responses here in one place (Module 34).

## Exercises

**Beginner.** `locals.requestId` (crypto.randomUUID) + log line per request with status and duration.

**Intermediate.** Session → `locals.user`; protect `/app/*` pages with redirect; prove `/api/webhooks/*` is unaffected (test it!).

**Production.** Middleware-based rate limiting for `POST` (in-memory for single instance; note distributed stores for serverless — Module 23) + security headers on every response.

**Architecture Challenge.** Multi-tenant: `locals.tenant` from custom domain. Where's the failure mode if middleware forgets a route?

> **Review:** never rely on route lists in middleware as the only check — the Action/endpoint/service must verify `resource.tenantId === locals.tenant.id` (Module 22). Middleware context = convenience + headers + logging; service layer = truth.

**Debugging Challenge.** `locals.user` is typed but `undefined` in an Action under serverless deployment. Session driver missing in production env (works locally with default fs driver). Fix: configure `session.driver` for the runtime / use the adapter's default; never assume dev drivers exist in prod.

## MENTAL MODEL — Middleware

> Middleware builds **request context and response policy** — who is asking (locals), how we log (request ID), what headers we send — before the route speaks. It is the *first* line of defense and the *worst* place to put the only line of defense.

## Official Documentation

- Middleware: https://docs.astro.build/en/guides/middleware/
- API reference (context): https://docs.astro.build/en/reference/api-reference/
- Sessions: https://docs.astro.build/en/guides/sessions/

## What I Should Know Before Continuing

1. Execution order: middleware vs page frontmatter vs action handlers.
2. How do you type `locals`? Why does it matter for refactoring safety?
3. Name three things middleware is *for* and three it is *not for*.
4. How would a middleware guard break a webhook route — and how do you prevent it?
