# Reference — Security Review Checklist (§71)

> Run at the end of **every major feature** and before every release. Evidence, not vibes: attach test names / curl output.

## A. Client boundary (frontend)
- [ ] No secrets in `dist/` bundles — CI grep for known prefixes (`sk_live`, `PRIVATE`, etc.)
- [ ] No server-only modules imported by islands or browser scripts
- [ ] `define:vars`/props contain only intentionally public data
- [ ] `set:html` never receives unsanitized user content
- [ ] Third-party scripts audited (need, size, consent)

## B. Server (Astro: middleware / actions / endpoints)
- [ ] Every input (body, params, query, headers used in logic) validated with Zod
- [ ] AuthN enforced (401) and AuthZ enforced per resource (403/404 policy: 404 for cross-tenant)
- [ ] Actions re-check authz inside handlers (middleware not the only gate)
- [ ] CSRF: `SameSite` cookies + origin checks on state-changing cookie-auth requests
- [ ] Webhooks: signature verified on raw body, timestamp tolerance, idempotent by event id
- [ ] Rate limits on auth, mutations, expensive reads (429 + `Retry-After`)
- [ ] Open-redirect protection on any `next`/`returnTo` params
- [ ] SSRF: outbound fetch targets allow-listed; uploads/images validated (magic bytes, pixel caps, re-encode)
- [ ] Error responses generic + `requestId`; stack traces never returned
- [ ] Security headers set: CSP, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`, HSTS (prod)

## C. Sessions & cookies
- [ ] Session cookie `HttpOnly; Secure; SameSite` (+ `__Host-` where possible)
- [ ] Session id rotated at login; revoked on logout/password change ("sign out everywhere")
- [ ] Absolute + idle expiration enforced server-side
- [ ] Passwords hashed with argon2/bcrypt via the auth library; login messages enumeration-safe
- [ ] Reset/verify tokens single-use, expiring, in POST bodies where possible

## D. Database
- [ ] All queries parameterized (no `sql.raw` with inputs)
- [ ] Tenant scoping in repository WHERE clauses + FKs on `org_id`
- [ ] Least-privilege DB role; prod credentials absent from dev/CI logs
- [ ] Sensitive columns encrypted/hashed as appropriate (passwords, tokens, PII policy)
- [ ] Migrations reviewed (no destructive drops without expand-contract)

## E. Infrastructure & supply chain
- [ ] Secrets in platform secret stores (runtime `process.env`), never images/config files
- [ ] `.env` gitignored; `.env.example` documented; secret scanning (gitleaks) in CI
- [ ] Dependencies audited (npm audit / Dependabot); lockfile committed
- [ ] Docker: minimal base, non-root, no secrets in layers, image scanning
- [ ] TLS everywhere; CORS default-deny with explicit origins
- [ ] Backups + restore tested; log retention/access policy set

## F. Tenancy & privacy
- [ ] Every tenant-owned query/mutation tested against **cross-tenant probes**
- [ ] Cache keys/headers can't leak personalized/tenant HTML (`private, no-store` when user-bound)
- [ ] Admin/impersonation actions audited with both identities recorded
- [ ] PII minimization in logs/analytics; consent honored for non-essential tracking

## Sign-off block

| Feature | Reviewer | Date | Evidence links | Result |
|---|---|---|---|---|
| | | | | PASS / FAIL |

**FAIL protocol:** fix → regression test added → re-review. A checklist without evidence is theater.
