# Module 15 — Server Endpoints

> Phase 13. Files in `src/pages/` can export HTTP verbs and return `Response` — your REST surface.

## Concept: an endpoint is a route that returns data, not HTML

```ts
// src/pages/api/posts.ts  →  GET /api/posts
import type { APIContext } from 'astro';
import { getCollection } from 'astro:content';

export function GET(_ctx: APIContext) {
  const posts = /* ... */;
  return Response.json(posts, {
    headers: { 'Cache-Control': 'public, max-age=60' },
  });
}
```

Export **named HTTP methods**: `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, `HEAD`. Dynamic params work like pages: `src/pages/api/posts/[id].ts` → `Astro.params.id`.

**Rendering:** endpoints prerender by default in `static` output (only GET makes sense); `export const prerender = false` for on-demand (needs adapter).

## Full CRUD endpoint

```ts
// src/pages/api/posts/[id].ts
import type { APIContext } from 'astro';
import { z } from 'astro/zod';

const UpdatePost = z.object({ title: z.string().min(1), body: z.string() });

export async function GET({ params, locals }: APIContext) {
  const post = await locals.services.posts.get(params.id!);
  if (!post) return new Response(null, { status: 404 });
  return Response.json(post);
}

export async function PUT({ params, request, locals }: APIContext) {
  if (!locals.user) return new Response(null, { status: 401 });
  const parsed = UpdatePost.safeParse(await request.json());
  if (!parsed.success) {
    return Response.json({ error: parsed.error.flatten() }, { status: 400 });
  }
  const post = await locals.services.posts.update(params.id!, parsed.data, locals.user.id);
  return Response.json(post);
}

export async function DELETE({ params, locals }: APIContext) {
  if (!locals.user) return new Response(null, { status: 401 });
  await locals.services.posts.delete(params.id!, locals.user.id);
  return new Response(null, { status: 204 });
}
```

## Streaming & files

```ts
export async function GET({ request }: APIContext) {
  // streaming JSON lines
  const stream = new ReadableStream({ /* ... */ });
  return new Response(stream, { headers: { 'Content-Type': 'application/x-ndjson' } });
}

// file download
return new Response(fileBuffer, {
  headers: {
    'Content-Type': 'application/pdf',
    'Content-Disposition': 'attachment; filename="report.pdf"',
  },
});
```

File **uploads** (`multipart/form-data`): `const form = await request.formData(); const f = form.get('file') as File;` — full treatment in Module 36.

## When endpoint vs Action vs form (preview of Module 16)

| Mechanism | Use |
|---|---|
| **Endpoint** | public/partner HTTP API, webhooks, feeds (RSS/JSON), file downloads, third-party consumption |
| **Action** | first-party mutations from your own UI (forms, islands) — typed, validated |
| **HTML form POST** | progressive enhancement target (can call an Action) |

If the consumer is *your* frontend → Actions. If the consumer is *anything else* → endpoints.

## Common Mistakes

**BAD:** returning `new Response('Internal error', { status: 500 })` after logging nothing.

**GOOD:** structured error log with request ID (Module 31) + generic client message.

**BAD:** GET endpoints with side effects (unsubscribe via GET link).

**GOOD:** POST for mutations; GET links only for idempotent reads.

**BAD:** `Access-Control-Allow-Origin: *` on authenticated APIs.

**GOOD:** explicit CORS allow-list (Module 23).

**BAD:** double frameworks — Express inside Astro for "real APIs."

**GOOD:** Astro endpoints are real APIs. (Node `middleware` mode exists if you must mount Astro inside an existing server.)

## Security Notes

- Validate **every** input with Zod (body, params, query).
- AuthN (`locals.user`) ≠ AuthZ (can *this* user touch *this* id?) — Module 22.
- Rate-limit mutating endpoints (Module 23).
- Set correct `Content-Type`; disable caching on personalized responses (`private, no-store`).

## Performance Notes

- JSON responses: compress (runtime/platform), paginate (Module 20).
- Connection reuse matters (Module 19).
- Cache public GETs via `context.cache` (Module 34).

## Exercises

**Beginner.** `GET /api/health` returning `{ ok: true, version }` with `no-store`.

**Intermediate.** `/api/posts` + `/api/posts/[id]` with Zod-validated `PUT`/`DELETE`, status codes 200/201/204/400/401/404.

**Production.** Add `ETag`/`If-None-Match` handling to `GET /api/posts` and pagination (`?cursor=`); test with curl.

**Architecture Challenge.** Mobile app + web app + Zapier need "get today's events." One endpoint versioned? Compare with: *a single `/api/v1/events` with cursor pagination + API key optional for partners + `Cache-Control` public for unauthenticated; Actions stay for the web UI; webhook out for Zapier triggers.*

**Debugging Challenge.** POST returns 404 from `static` build: the endpoint was prerendered as a static file (only GET exists in `dist`). Add `export const prerender = false` + adapter, or set `output: 'server'`.

## MENTAL MODEL — Endpoints

> An endpoint is a **route that speaks HTTP instead of HTML**. Same router, same middleware, same locals — different representation. Treat status codes, validation, and caching as part of the interface; consumers other than your UI are first-class.

## Official Documentation

- Endpoints: https://docs.astro.build/en/guides/endpoints/
- APIContext reference: https://docs.astro.build/en/reference/api-reference/
- Routing: https://docs.astro.build/en/guides/routing/

## What I Should Know Before Continuing

1. How do endpoint files map to URLs and verbs?
2. Endpoint vs Action: the decision rule.
3. How do you upload files and stream responses?
4. Why is GET-with-side-effects an anti-pattern, and what breaks around it (crawlers, prefetch)?
