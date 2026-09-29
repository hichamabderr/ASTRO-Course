# Module 20 — Data Fetching & API Design

> Phases 30–31. Six ways to get data in Astro — where each runs, what serializes to the browser, and how to design HTTP APIs if you must expose one.

## The six approaches (compare on one axis)

| Approach | Runs where | Who calls it | What's serialized to browser | Use when |
|---|---|---|---|---|
| **Content queries** (`getCollection`) | build/server | your pages | just rendered HTML | all editorial content |
| **Direct server fetch/DB in frontmatter** | build or request | your pages/actions | just HTML/DTO | data needed to *render* |
| **Server island** (`server:defer`) | request (deferred) | runtime fragment route | HTML fragment | personalized/slow fragments in static/SSR pages |
| **Action** | request | your forms/islands | typed JSON result | first-party mutations & typed reads |
| **Endpoint** | request | external clients, feeds | JSON/bytes | public API, webhooks |
| **Client fetch** | browser | islands | — (request only) | user-driven queries after load |

**The serialization question is the discipline:** everything fetched in frontmatter leaves as *HTML*; everything fetched from an island lands as *bytes in the browser*. Choose the earliest server location that can do the job.

## Patterns worth stealing

```astro
---
// 1. Server-side parallel fetch in frontmatter (prerender or SSR)
const [settings, testimonials] = await Promise.all([
  getEntry('settings', 'site'),
  getCollection('testimonials', ({ data }) => data.featured),
]);
---
```

```ts
// 2. Action read for island refreshes (typed!)
export const server = {
  notifications: defineAction({
    handler: async (_, ctx) => {
      if (!ctx.locals.user) throw new ActionError({ code: 'UNAUTHORIZED' });
      return ctx.locals.services.notifications.recent(ctx.locals.user.id);
    },
  }),
};
```

Client-side fetch to *your own* first-party domain should usually be an **action call** (typed + validated). Client fetch is for *other* origins and streaming UIs.

## API design fundamentals (for your endpoints)

**Resources & verbs:**

```text
GET    /api/v1/posts            list (paginated, filterable)
GET    /api/v1/posts/:id        read
POST   /api/v1/posts            create → 201 + Location
PATCH  /api/v1/posts/:id        partial update
DELETE /api/v1/posts/:id        → 204
```

**Status codes (the ones you actually use):**
`200` OK · `201` created · `204` no content · `400` bad input · `401` unauthenticated · `403` unauthorized · `404` not found · `409` conflict · `422` validation detail (if you distinguish) · `429` rate limited · `500` server fault.

**Error format (consistent, boring):**

```json
{ "error": { "code": "VALIDATION_FAILED", "message": "Human readable", "fields": { "email": "Invalid" }, "requestId": "…" } }
```

**Pagination/filtering/sorting:**

```text
GET /api/v1/posts?limit=20&cursor=abc&tag=astro&sort=-publishedAt
```
- Cursor for large/streaming sets; `page`/`limit` acceptable for admin tables.
- Allow-list `sort` fields (never raw column names from query strings).

**Versioning:** `/api/v1/` prefix for anything external; additive changes preferred; deprecation headers.

**Idempotency:** mutating endpoints that clients retry (payments, webhooks) accept `Idempotency-Key` (Module 36).

**Webhooks (inbound):** signature verification + timestamp tolerance + idempotent handling (Module 36).

## Common Mistakes

**BAD:** `useEffect` fetch of a config object that never changes per user.

**GOOD:** frontmatter/static — or inline it.

**BAD:** BFF endpoint that just proxies your DB to your own island with no validation.

**GOOD:** action with Zod input (it *is* the typed BFF).

**BAD:** `GET /api/deleteAccount?confirm=1`.

**GOOD:** `POST`/`DELETE` verbs for mutations (prefetch safety).

**BAD:** leaking `total` pagination math on huge tables (`COUNT(*)` per request).

**GOOD:** `hasMore` via limit+1.

## Security Notes

- Public GETs are attack surface too: validate query params, cap `limit`, allow-list sorts/filters.
- CORS default-deny; credentials only to your origins (Module 23).
- Never return stack traces; include `requestId` (Module 31).
- Cache only **anonymous, non-personal** responses publicly.

## Performance Notes

- Frontmatter fetches at build = free at runtime; frontmatter fetches in SSR = latency — batch & cache (Module 34).
- Islands: avoid waterfall fetches; one action call returning a composite DTO beats three.
- Compress JSON (platform), keep DTOs lean.

## Exercises

**Beginner.** Render a page from a public API fetched in frontmatter (prerendered with `getStaticPaths` or one SSR page) — zero client JS.

**Intermediate.** `/api/v1/events` endpoint: cursor pagination, `?category=` filter allow-list, error envelope, `429` after 60 req/min.

**Production.** Replace a client-side `fetch('/api/me')` pattern in an island with an action call; measure DX (types) and bytes.

**Architecture Challenge.** Dashboard island needs: user, 3 metrics, notification list. One round trip or four?

> **Review:** one action `getDashboardSnapshot()` returning a typed composite — fewer RTTs, one authz check, one DTO. Break apart only if parts cache/refresh differently (then `server:defer`/polling per widget).

**Debugging Challenge.** Build fails only in CI: frontmatter `fetch()` to localhost API. You fetch your own endpoints at build time — but the API isn't running in CI. Fix: call the service/DB directly (server-side) instead of HTTP-looping yourself.

## MENTAL MODEL — Data Fetching

> Data should arrive at the **latest possible server stage** that still renders correctly: build-time if it can freeze, request-time if it must be fresh, deferred if it's slow, client-side only if the *user* drives it. HTTP APIs are for external consumers; actions are for your UI.

## Official Documentation

- Data fetching: https://docs.astro.build/en/guides/data-fetching/
- Endpoints: https://docs.astro.build/en/guides/endpoints/
- Actions: https://docs.astro.build/en/guides/actions/
- Live collections: https://docs.astro.build/en/guides/content-collections/#live-content-collections

## What I Should Know Before Continuing

1. The six approaches table — fill it from memory with the serialization column.
2. Compose an error envelope and status-code set for a small API.
3. When should an island fetch vs receive props?
4. Why is an action better than a hand-rolled BFF endpoint for your own UI?
