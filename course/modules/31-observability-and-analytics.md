# Module 31 — Observability & Analytics

> Phases 27/61–62. Structured logging, request IDs, errors, Web Vitals, privacy-aware analytics — instrumented the Astro way.

## Concept: you can't operate what you can't see

```text
Request ─► middleware assigns requestId ─► logs (structured) ─► your log stack
                                     └─► error reports with requestId
Browser ─► Web Vitals beacons ─► analytics endpoint ─► metrics
```

## Structured logging (Astro v7 `logger` is stable)

```ts
// src/middleware.ts excerpt
const logging = defineMiddleware(async (ctx, next) => {
  const requestId = crypto.randomUUID();
  ctx.locals.requestId = requestId;
  const start = performance.now();
  try {
    const response = await next();
    log('info', 'request', {
      requestId, path: ctx.url.pathname, status: response.status,
      ms: Math.round(performance.now() - start),
    });
    return response;
  } catch (err) {
    log('error', 'request', { requestId, path: ctx.url.pathname, err: String(err) });
    throw err;
  }
});
```

Rules: **JSON lines**, stable field names, **never log secrets** (tokens, cookies, passwords), include `requestId` everywhere. Astro 7's structured `logger` config (top-level `logger` in `astro.config.mjs`) can standardize dev/build output — server runtime logs are yours.

## Error handling

- Actions: `ActionError` for expected; unexpected errors bubble → middleware → 500 + `requestId` in the response body/message.
- Pages: custom `500.astro`; SSR `Astro.rewrite`/adapter error hooks for pretty errors.
- Report to Sentry/etc. **server-side** with scrubbing; client-side error hooks only in islands.

## Web Vitals & real-user monitoring

```ts
// tiny island or idle script — send on visibility-hidden
import { onLCP, onINP, onCLS } from 'web-vitals';
for (const fn of [onLCP, onINP, onCLS]) {
  fn((m) => navigator.sendBeacon('/api/vitals', JSON.stringify({
    name: m.name, value: m.value, path: location.pathname,
  })));
}
```

Endpoint batches into your metrics store. Lab (Lighthouse) is for debugging; **field data** is for decisions (Module 29).

## Analytics: pageviews, events, privacy

| Approach | Notes |
|---|---|
| Server-side pageviews | from SSR logs/static host analytics — zero client JS |
| Privacy-first SaaS | Plausible, Cloudflare Web Analytics, etc. — lightweight scripts |
| Product analytics | event API via actions/endpoints; consent-gated |

**Consent & cookies:** EU-style regimes need consent before non-essential cookies/trackers. Patterns: consent banner (island!), conditional script loading, server-side analytics where lawful. **Performance cost** of tag managers is notorious — budget third-party JS (Module 29) and use facades (Module 36's embed pattern).

## Tracing concepts

- Propagate `x-request-id` through service calls; log spans (start/end) around DB/HTTP.
- Distributed tracing (OpenTelemetry) is straightforward in Node standalone; constrained on edge runtimes.

## Common Mistakes

**BAD:** `console.log(user)` in middleware (leaks PII into log drains).

**GOOD:** ids + outcomes, scrubbed.

**BAD:** client-only error tracking that misses SSR failures.

**GOOD:** server error reporting + client beacons in islands.

**BAD:** five analytics scripts + tag manager, all sync in `<head>`.

**GOOD:** one privacy-aware source, async/idle, consent-correct.

## Security Notes

- Logs are sensitive data stores: access control + retention policies.
- Error pages/500s must include `requestId` but **no stack traces**.
- Vitals endpoint: rate-limit, validate payload shape (public endpoint!).

## Performance Notes

- `sendBeacon` on hide — no impact on load.
- Sampling (1–10%) is normal at scale.

## Exercises

**Beginner.** Request-ID middleware + JSON logs; find a request end-to-end in logs.

**Intermediate.** `/api/vitals` endpoint + `web-vitals` beacon; dashboards for p75 LCP.

**Production.** Sentry (or similar) with scrubbing + release tagging; alert on 5xx rate by route.

**Architecture Challenge.** Product wants session replay + 4 marketing tags + product analytics on a content site. Cost it against the performance budget.

> **Review:** replay only on app zone with consent; collapse marketing tags into one privacy-first analytics + server-side events; product analytics via one beacon endpoint. Total third-party ≤ 20 KB or rejected by budget.

**Debugging Challenge.** 500s spike after deploy but logs show nothing. Error handler swallowed errors before logging (`catch { return new Response('err') }`). Fix: log first with requestId, then respond; add a 5xx alert.

## MENTAL MODEL — Observability

> Every request gets an **identity** (requestId); every outcome gets a **structured line**; every user-visible delay gets a **metric**. If a feature ships without logs, metrics, and an error story, it isn't done — and you'll debug it blind at 3 AM.

## Official Documentation

- Logging (v7 logger): https://docs.astro.build/en/reference/configuration-reference/#logger
- Middleware: https://docs.astro.build/en/guides/middleware/
- Web Vitals: https://github.com/GoogleChrome/web-vitals

## What I Should Know Before Continuing

1. What goes in a structured request log line? What never does?
2. Lab vs field Web Vitals — which decides product direction?
3. Where do SSR errors get reported vs island errors?
4. Three performance costs of tag managers; three privacy costs of trackers?
