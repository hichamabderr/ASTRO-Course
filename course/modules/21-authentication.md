# Module 21 — Authentication

> Phase 16. **Astro has no single official auth system** — by design (verified: [Authentication guide](https://docs.astro.build/en/guides/authentication/)). This module teaches the *concepts* rigorously, then implements one primary architecture: **session cookies + Better Auth**.

## Concept: authentication vs authorization

- **Authentication (AuthN):** *who are you?* → identity (login, OAuth, passkeys).
- **Authorization (AuthZ):** *what may you do?* → permissions (Module 22).

Never mix them: being logged in proves nothing about which rows you may touch.

## The landscape (verified, Sep 2026)

| Option | Model | Notes |
|---|---|---|
| **Better Auth** (course primary) | framework-agnostic TS auth framework; sessions, OAuth, plugins (2FA, orgs…) | official docs' lead example; mounts on an Astro endpoint; works with your DB |
| **Clerk** | hosted UI + APIs | official Astro SDK; fastest DX; vendor cost/lock-in |
| **Supabase Auth** | hosted (GoTrue) | good with Supabase DB/RLS stacks |
| **Auth.js / community** | adapters exist | evaluate maintenance |
| **Lucia** ⛔ | was a session-auth learning resource | **not a maintained library anymore** — its site teaches the patterns; old tutorials referencing "the Lucia package" are stale |
| Roll your own | educational only | you will reimplement reset/rotation/MFA badly — do it once to learn, never in prod |

## The architecture (primary)

```text
Browser ──POST /api/auth/*──► Better Auth handler (endpoint) ──► DB (users, sessions)
   ▲                                    │
   │ HttpOnly session cookie            ▼
   └────────────────── middleware verifies ──► locals.user / locals.session
                                                     │
                              Actions/pages re-check ──► services enforce ownership
```

### 1. Mount the auth handler

```ts
// src/pages/api/auth/[...all].ts  (catch-all endpoint)
import { auth } from '../../server/auth';   // Better Auth instance
import type { APIRoute } from 'astro';

export const ALL: APIRoute = ({ request }) => auth.handler(request);
// (Check Better Auth's current Astro integration docs for the exact export shape.)
```

### 2. Middleware → `locals`

```ts
// src/middleware.ts (excerpt)
import { defineMiddleware } from 'astro:middleware';
import { auth } from './server/auth';

export const onRequest = defineMiddleware(async (ctx, next) => {
  const session = await auth.api.getSession({ headers: ctx.request.headers });
  ctx.locals.user = session?.user ?? null;
  ctx.locals.session = session?.session ?? null;
  return next();
});
```

### 3. Guard pages/actions — **re-check inside handlers** (Module 22)

## Sessions & cookies (the real curriculum)

| Property | Meaning | Setting |
|---|---|---|
| `HttpOnly` | JS can't read the cookie (XSS can't exfiltrate session id as easily) | always for session cookies |
| `Secure` | sent only over HTTPS | always in prod |
| `SameSite=Lax/Strict` | CSRF mitigation for cross-site navigation | `Lax` typical; `Strict` for high-security |
| Session id | random, server-side record (or signed token) | rotate on login |
| Expiration | absolute + idle timeouts | server-enforced |
| Revocation | logout / "sign out everywhere" / password change | delete/flag session rows |

**Astro Sessions** (stable) can store *your* session state (`context.session`) with pluggable drivers; auth libraries like Better Auth manage their own session tables. Either way: **the browser holds an opaque id; the server holds the truth.**

## The flows you must implement (and test)

1. **Register** — validate, hash password (argon2/bcrypt via the library), verify email.
2. **Login** — constant-time failures ("invalid credentials" generic), session creation, **rotation**.
3. **Logout** — destroy server session + clear cookie (`Max-Age=0`).
4. **Password reset** — single-use, expiring tokens; never reveal account existence; token in POST body not GET logs.
5. **Email verification** — signed, expiring links.
6. **OAuth** — `state`/PKCE; account linking rules.
7. **MFA (concept)** — TOTP/WebAuthn as second factor; recovery codes; step-up for sensitive ops.

## OLD vs MODERN pitfalls

| Old tutorial says | Reality |
|---|---|
| store JWT in `localStorage` | vulnerable to XSS; prefer HttpOnly cookies |
| "Lucia the npm library" | Lucia is a learning resource now — use Better Auth/etc. |
| `astro login` / Astro Studio auth | CLI commands removed in v7 |
| middleware-only protection | handlers must enforce too |
| password `md5`/`sha256` | use argon2/bcrypt via the auth lib |

## Common Mistakes

**BAD:** `fetch('/api/me')` on every island to know who the user is.

**GOOD:** server renders user-dependent HTML; islands receive `user={{id,name}}` props.

**BAD:** user object (with role) kept in a client store as the "source of truth."

**GOOD:** client mirrors display state; every mutation re-authorized server-side.

**BAD:** session cookie without `Secure` "because staging is HTTP."

**GOOD:** platform-level HTTPS everywhere; HSTS in prod (Module 23).

**BAD:** timing-unsafe password compares / user enumeration via error messages.

**GOOD:** library-grade implementations — this is *why* Better Auth exists.

## Security Notes

- Auth cookies are the crown jewels: `HttpOnly; Secure; SameSite`, short idle TTL, rotation, revocation.
- CSRF: `SameSite` + origin checks on state-changing requests (Module 23) — belt and suspenders.
- Rate-limit login/reset endpoints (Module 23).
- Log auth events (login, reset, revoke) with request IDs — no secrets in logs.

## Performance Notes

- Session lookup per request: cache in-process briefly or use fast stores (Module 34); don't run 3 queries per asset-less request.
- Auth libraries add server bundle weight — fine for SSR routes; keep them out of prerendered pages (they can't run there anyway).

## Exercises

**Beginner.** Register/login/logout with Better Auth (or a mock service) on the Node adapter; show `locals.user` in an SSR page header.

**Intermediate.** Password reset flow end-to-end with expiring tokens and enumeration-safe messages (intercept emails in dev with a mailcatcher).

**Production.** Add email verification + "revoke all sessions on password change" + audit log rows; write Playwright tests for each flow (Module 30).

**Architecture Challenge.** SaaS with org accounts, Google OAuth, and an admin impersonation feature. Sketch session model + risks.

> **Review:** sessions carry `userId + orgId + role`; impersonation = audited, admin-only action creating a *scoped, short-TTL* session with `impersonatedBy` recorded; OAuth linking requires verified-email matching policy; every session change is logged.

**Debugging Challenge.** Works on `localhost`, logged out forever on production domain. Cookie `Secure` + `SameSite=None` misconfig, or domain mismatch (`__Host-` prefix rules), or missing HTTPS. Inspect `Set-Cookie` in devtools; fix attributes per the table above.

## MENTAL MODEL — Authentication

> Authentication is **outsourced judgment**: a library proves identity, the server records it in a session, middleware surfaces it as `locals`, and every handler still asks "is this allowed?" The cookie is a claim, not a fact — the database is the fact.

## Official Documentation

- Astro authentication guide: https://docs.astro.build/en/guides/authentication/
- Sessions: https://docs.astro.build/en/guides/sessions/
- Better Auth docs: https://www.better-auth.com/docs/ (verify Astro integration section)
- Clerk Astro SDK: https://clerk.com/docs (Astro section)

## What I Should Know Before Continuing

1. AuthN vs AuthZ — and why login ≠ permission.
2. Cookie attribute matrix: what each of `HttpOnly/Secure/SameSite` defends against.
3. The seven flows; which tokens are single-use and why.
4. Why is JWT-in-localStorage considered legacy advice?
