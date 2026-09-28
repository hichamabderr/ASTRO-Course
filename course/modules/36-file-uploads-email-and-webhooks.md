# Module 36 — File Uploads, Email & Webhooks

> Phases 53–55. Three integration surfaces that every full-stack app meets — each with sharp security edges.

## Part 1 — File uploads

### The flow

```text
<form enctype="multipart/form-data">  →  Action/endpoint (z.instanceof(File))
   → validate (type, size, pixels)  →  re-encode  →  object storage (S3/R2/Supabase)
   → store URL/key in DB            →  serve via signed URLs / CDN
```

```ts
// action excerpt (accept: 'form')
input: z.object({
  file: z.instanceof(File)
    .refine((f) => f.size <= 5 * 1024 * 1024, 'Max 5MB')
    .refine((f) => ['image/png', 'image/jpeg', 'image/webp'].includes(f.type), 'Images only'),
}),
handler: async ({ file }, ctx) => {
  const buf = Buffer.from(await file.arrayBuffer());
  const meta = await probeImage(buf);          // real dimensions / decompression-bomb check
  const safe = await reencode(buf);            // strips EXIF/payloads
  const key = `uploads/${crypto.randomUUID()}.webp`;
  await storage.put(key, safe, { contentType: 'image/webp' });
  return { url: key };
}
```

### Rules (non-negotiable)

1. **Validate content, not filename/extension** (`file.type` is client-declared! sniff magic bytes).
2. **Size caps** at proxy + handler; **pixel caps** (decompression bombs).
3. **Re-encode images** through sharp/image service — strips EXIF (privacy) and polyglots.
4. Store **outside** the web root; serve via **signed URLs** or CDN with auth-aware access (Module 22 tenant checks on the lookup).
5. Untrusted documents (PDFs/docx): sandbox processing; AV scanning if the threat model needs it.

## Part 2 — Email

Use cases: verification, password reset, welcome, notifications, billing receipts.

```ts
// src/server/email.ts — provider SDK (Resend/SES/Postmark/SMTP) — SERVER ONLY
export async function sendPasswordReset(to: string, token: string) {
  await provider.send({
    to, from: 'no-reply@example.com',
    subject: 'Reset your password',
    html: `<a href="https://example.com/reset?token=${token}">Reset</a>`,
    text: `Reset: https://example.com/reset?token=${token}`,
  });
}
```

Rules:
- **Credentials live in `process.env`** (Module 33) — never islands.
- Tokens in email links are **single-use + expiring** (Module 21).
- Queue heavy/bulk sends; don't block request handlers on SMTP.
- Templates: render server-side (Astro can render `.astro` to HTML strings via the Container API, or simple template literals).
- Dev: mailcatcher/Mailpit to avoid sending real mail.

## Part 3 — Webhooks (inbound)

```ts
// src/pages/api/webhooks/stripe.ts
import type { APIContext } from 'astro';

export async function POST({ request, locals }: APIContext) {
  const raw = await request.text();
  const sig = request.headers.get('stripe-signature')!;
  const event = verifyStripeSignature(raw, sig, process.env.STRIPE_WEBHOOK_SECRET!);  // throws if invalid

  // idempotency: providers retry!
  if (await alreadyProcessed(event.id)) return new Response(null, { status: 200 });

  switch (event.type) {
    case 'checkout.session.completed': await orders.fulfill(event.data); break;
    // ...
  }
  await markProcessed(event.id);
  return new Response(null, { status: 200 });
}
```

### Webhook security checklist

1. **Signature verification** with raw body (before JSON parse) + timestamp tolerance (replay defense).
2. **Idempotency** — store event ids; retried deliveries must not double-charge.
3. **Return 2xx fast**; process async if slow (queues) — providers time out.
4. **Log everything** (event id, type, result) with request IDs.
5. Webhook routes are **CSRF-exempt** (they use signatures, not cookies) — but they are *not* auth-exempt (Module 23's debugging challenge).
6. **Outbound webhooks** (you → customers): sign payloads, exponential-backoff retries, dead-letter queues.

## Common Mistakes

**BAD:** trusting `file.name` for storage paths (path traversal, collisions).

**GOOD:** server-generated keys (`uuid.webp`).

**BAD:** sending email synchronously in a login action.

**GOOD:** enqueue; user sees "check your inbox" instantly.

**BAD:** webhook handler doing 30s of image processing inline.

**GOOD:** acknowledge + queue.

**BAD:** `GET /unsubscribe?email=…` (CSRF-able, logged in proxies).

**GOOD:** signed token POST/GET with `List-Unsubscribe` standards compliance.

## Security Notes

- Uploads: XSS via SVG (`<script>` in SVG) — serve SVGs with `Content-Disposition: attachment` or sanitize/re-rasterize.
- Email header injection — use provider SDKs, never string-concatenate headers.
- Webhook secrets are runtime `process.env`.

## Performance Notes

- Uploads: presigned-direct-to-storage patterns bypass your server bandwidth (advanced; verify handler still enforces policy).
- Mail: async queues; batch digests.

## Exercises

**Beginner.** Avatar upload (5MB, png/jpeg/webp) via action → local disk (dev) → object storage (prod); validate via magic bytes.

**Intermediate.** Password-reset email through a dev mailcatcher; tokens single-use; test replay rejection.

**Production.** Stripe-style webhook with signature verify + idempotency table + structured logs; chaos-test with duplicate deliveries.

**Architecture Challenge.** User-generated video platform: upload, transcode, moderation. Where does Astro sit?

> **Review:** Astro Actions handle upload initiation (auth + policy + signed upload URL); workers transcode outside the request; moderation queue is server-side; Astro serves the app + catalog (cached); never transcode in a request handler.

**Debugging Challenge.** Webhook works in Stripe CLI but 400s in prod: handler `await request.json()` before verifying → signature over parsed/re-serialized body mismatch (or body already consumed). Fix: `request.text()` first, verify over raw string, then `JSON.parse`.

## MENTAL MODEL — Integrations

> Uploads, email, and webhooks are **trust boundaries with latency**: validate everything inbound (bytes, signatures), move slow work off the request path, and make every delivery **idempotent** — because the network will retry, and users will double-click.

## Official Documentation

- FormData/actions file inputs: https://docs.astro.build/en/guides/actions/
- Endpoints: https://docs.astro.build/en/guides/endpoints/
- OWASP file upload: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html

## What I Should Know Before Continuing

1. Why is `file.type` not a security control? What is?
2. Draw the upload pipeline with its four validation gates.
3. Webhook checklist: signature, idempotency, timeout, logs, CSRF exemption.
4. Why must password-reset tokens be single-use and expiring?
