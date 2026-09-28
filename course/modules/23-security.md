# Module 23 — Security (Comprehensive)

> Phases 26–27 & 71. The full threat tour for an Astro stack — separated by **layer** (frontend / server / database / infrastructure) — plus the recurring **security review checklist**.

## Layered responsibilities

| Layer | Owns |
|---|---|
| **Frontend (HTML/islands)** | output encoding (default), no secrets, safe URL handling, CSP compatibility |
| **Server (Astro: middleware/actions/endpoints)** | input validation, authn/authz, CSRF, headers, rate limits, safe errors |
| **Database** | parameterized queries, least privilege, RLS/tenant scoping, encryption at rest |
| **Infrastructure** | TLS, secrets management, WAF/CDN, dep hygiene, backups |

## The threat tour

### XSS (Cross-Site Scripting)
- **Risk:** injected script executes in your origin.
- **Astro specifics:** `{expr}` **auto-escapes**; `set:html` **does not**. MDX is code — never render untrusted MDX. `define:vars` publishes values to scripts.
- **Defense:** sanitize (allow-list) before `set:html`; CSP (`script-src`); `HttpOnly` cookies limit session theft.

### CSRF (Cross-Site Request Forgery)
- **Risk:** attacker site triggers authenticated POSTs.
- **Defense:** `SameSite=Lax/Strict` cookies; verify `Origin`/`Sec-Fetch-Site` on state-changing requests in middleware; custom header requirement for action/fetch calls; re-auth for high-stakes ops.

```ts
// middleware excerpt
if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(ctx.request.method)) {
  const origin = ctx.request.headers.get('origin');
  if (origin && !allowedOrigins.has(origin)) return new Response(null, { status: 403 });
}
```

### SSRF (Server-Side Request Forgery)
- **Risk:** server fetches attacker-chosen URLs (webhooks, image proxies, importers).
- **Defense:** allow-list outbound hosts; block private IP ranges; timeouts + size caps; no user-controlled URLs in server fetch without validation. (Cf. image `domains`/`remotePatterns` config — Module 25.)

### Open redirects
- `return redirect(url.searchParams.get('next'))` → phishing vector.
- **Defense:** allow-list paths (`startsWith('/') && !startsWith('//')`) or fixed hosts.

### Session & cookie security
Covered in Module 21: `HttpOnly; Secure; SameSite`, rotation, revocation, timeouts.

### Authorization bypass / IDOR
Module 22: scoped queries + service checks; 404 for cross-tenant.

### SQL injection
Parameterize **everything** (Drizzle/`postgres-js` do). Never `sql.raw(userInput)`. LIKE-escape user patterns.

### Input validation & output encoding
Zod at every boundary (bodies, params, queries, headers used in logic). Encode on output per context (HTML/attr/URL/JS).

### File uploads
Module 36: type sniffing (not extension), size caps, re-encode images, storage outside web root, signed URLs, AV scanning for untrusted sources.

### Secrets management
Module 33: `process.env` runtime secrets; platform secret stores; never `PUBLIC_`; never client islands.

### CORS
Default-deny; explicit origins; `Access-Control-Allow-Credentials` only for your exact origins; preflight caching sane.

### Security headers & CSP

```ts
// middleware sets these on every response
const csp = [
  "default-src 'self'",
  "script-src 'self'",              // tighten per your islands' needs
  "style-src 'self' 'unsafe-inline'", // Astro scoped styles are inline-ish; evaluate nonces if strict
  "img-src 'self' data: https://cdn.example.com",
  "connect-src 'self'",
  "frame-ancestors 'none'",
  "base-uri 'self'",
].join('; ');
response.headers.set('Content-Security-Policy', csp);
response.headers.set('X-Content-Type-Options', 'nosniff');
response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
response.headers.set('Permissions-Policy', 'camera=(), geolocation=()');
// HSTS in prod: 'max-age=31536000; includeSubDomains'
```

(Astro 6 stabilized CSP support — check current docs for built-in helpers vs hand-rolled headers.)

### Rate limiting
Middleware/proxy counters per IP+route for login, resets, mutations, expensive queries. 429 with `Retry-After`. Distributed stores (Redis/KV) on serverless.

### Dependency security
Lockfile + `npm audit`/Dependabot in CI; pin majors; watch advisories for auth/ORM/image libs; `npx @astrojs/upgrade` for framework integrations.

## The Security Review Checklist (run at the end of every feature)

1. Can the client see this secret? (bundle/HTML/props/`define:vars`)
2. Is authorization enforced **server-side**, per resource?
3. Can user A reach user B's resource by changing an ID?
4. Are cookies `HttpOnly/Secure/SameSite`? Sessions rotated/revocable?
5. Is every endpoint/action protected against CSRF & rate abuse?
6. Are webhooks signature-verified and idempotent?
7. Is every input validated (Zod) at the boundary?
8. Is every output encoded for its context? Any `set:html` with user data?
9. Are uploads validated, capped, and stored safely?
10. Is tenant isolation enforced in **queries**, not just UI?
11. Do error responses leak internals? Are logs free of secrets?
12. Are security headers/CSP set and tested?

Print this. Use it in PR reviews (Module 41 also references it).

## Common Mistakes (security edition)

**BAD:** security through hidden UI.
**GOOD:** server-side predicates.

**BAD:** `dangerouslySetInnerHTML`-style habits (`set:html`) for "just a title."

**GOOD:** escaped expressions; sanitized HTML only where content demands.

**BAD:** trusting `X-Forwarded-For` blindly for rate limiting.

**GOOD:** trust only your proxy's header configuration.

**BAD:** one giant admin JWT in localStorage.

**GOOD:** server sessions, scoped, revocable.

## Performance Notes

- CSP/headers are free. Rate limiting costs a lookup — do it on mutations first.
- Sanitization runs server-side at render/build where possible (once), not in the browser per paint.

## Exercises

**Beginner.** Add the security headers block above to middleware; verify with `curl -I`.

**Intermediate.** Deliberately create an XSS via `set:html` with an "author bio" field; fix it with sanitization; add a test.

**Production.** Origin-check + rate-limit middleware for all POST routes; Playwright tests for CSRF rejection.

**Architecture Challenge.** A "safe HTML" feature letting marketing paste HTML into pages. Design the sandbox.

> **Review:** allow-list sanitizer (tags/attrs/URL protocols) server-side at save + at render; no `<script>`/`on*`/`javascript:`; CSP as second layer; preview in isolated iframe (`sandbox`) for editors.

**Debugging Challenge.** Webhook works in curl, 403 in production: your CSRF origin check rejects signature-verified webhook POSTs (no `Origin`/wrong origin). Fix: exempt webhook routes (they use signature auth), or gate the check on cookie-authenticated requests only.

## MENTAL MODEL — Security

> Security is **layered skepticism**: escape by default, validate at boundaries, authorize next to data, isolate tenants in queries, hold secrets server-side, and verify the strangers (webhooks, uploads) twice. Every feature ships with its checklist pass — not a quarterly surprise.

## Official Documentation

- Astro security guide: https://docs.astro.build/en/guides/security/
- Middleware: https://docs.astro.build/en/guides/middleware/
- OWASP Top 10: https://owasp.org/www-project-top-ten/
- Cheat sheets: https://cheatsheetseries.owasp.org/

## What I Should Know Before Continuing

1. Which layer owns CSP? Parameterized queries? Origin checks?
2. Walk the 12-point checklist against the newsletter form you built in Module 17.
3. Two XSS vectors specific to Astro templates. How does `set:html` differ from `{}`?
4. Why do webhooks skip CSRF checks — and what replaces them?
