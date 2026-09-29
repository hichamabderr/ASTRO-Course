# Project 3 — Full-Stack SaaS ("Acme Cloud" — marketing + portal)

> Modules applied: 14–24, 30, 33–38. Deliverable: the capstone's engineering core: auth, tenancy, Actions, DB, SSR/static mix, testing, deployment.

## Feature list

### Public (static)
- Landing, pricing, blog (reuse Projects 1–2), docs stub

### Auth
- Register / login / logout (Better Auth — Module 21), email verification, password reset
- Session cookies hardened (`HttpOnly; Secure; SameSite`), revocation on password change

### Customer portal (SSR)
- Dashboard (metrics + notifications island with polling — Module 24's TanStack Query *or* light polling)
- Profile + settings (Actions: `updateProfile`)
- Billing page (plan + invoices; fake Stripe or test mode)
- File upload avatar (Module 36 rules)

### Admin (SSR + strict RBAC)
- Users list, organizations, role management
- Audit log viewer

### System
- Middleware: requestId, security headers, session → `locals.user`/`locals.tenant`
- Actions for all first-party mutations; one public endpoint (`/api/health`) + one webhook (signature-verified)
- PostgreSQL + Drizzle with migrations; seeded test data
- Testing pyramid (Module 30) including **authz matrix**
- Docker deploy (Module 37) or Node standalone; env validated at boot (Module 33)
- Logging + request IDs (Module 31)

## Architecture (the one to implement)

```text
src/
  pages/                 index, pricing, blog/**            (prerender)
                         app/**  admin/**                   (prerender = false)
                         api/health.ts  api/webhooks/stripe.ts
                         api/auth/[...all].ts
  actions/               auth-safe ops, profile, billing, admin
  middleware.ts          logging → security → session context
  server/
    auth/  db/ (schema, client, migrations)  services/  repositories/  permissions/
  components/react/      DashboardWidgets, Notifications, UploadAvatar (islands)
```

**Render strategy table (submit this as part of the project):**

| Route | Mode | Cache |
|---|---|---|
| /, /pricing, /blog/** | static | immutable assets |
| /app/** | SSR | `private, no-store` |
| /admin/** | SSR | `no-store` |
| /api/health | SSR | `no-store` |
| dashboard metrics fragment | SSR/server:defer | short swr per user? no → no-store |

## Milestones

1. **M1 — DB + layering:** schema, migrations, repositories, services with tests (19).
2. **M2 — Auth:** Better Auth mounted; middleware locals; guarded pages (21).
3. **M3 — Portal:** settings actions with validation + a11y forms (16–17).
4. **M4 — Tenancy:** orgs/roles; IDOR-proof queries; authz matrix tests (22, 30).
5. **M5 — Ops:** uploads, webhook, logs, env validation, Docker (31, 33, 36–37).
6. **M6 — Hardening:** security checklist pass (Module 23's 12 points), load test one page.

## Definition of done

- [ ] Unauthenticated `/app/*` → login redirect; direct action calls while logged out → 401
- [ ] Cross-tenant resource IDs → 404 (test it!)
- [ ] `dist/` contains no secrets (CI grep test)
- [ ] All migrations reproducible on a fresh DB; seed works
- [ ] `astro check` + unit + integration + Playwright smoke green in CI
- [ ] Webhook idempotency test passes (duplicate event delivered twice)

## Stretch

- Notifications polling island with TanStack Query in an app zone (Module 24)
- `routeRules` caching on one public SSR route with tag invalidation from an action
- Rate limiting on login + password reset

## Security review (graded — use Module 23's checklist)

Submit a filled checklist with evidence (test names, curl outputs, screenshots).
