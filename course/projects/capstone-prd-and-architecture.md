# Capstone — "Modern SaaS + Documentation + Content Platform"

> **This document is a PRD, not an implementation.** Per the course brief (§97): read the PRD, **design the solution yourself first** (the 11 design tasks below), then study the review section. That review is the final exam.

---

## 1. Product Requirements Document

### 1.1 Product summary
**Acme Cloud** (fictional): a developer-tools SaaS with a public content platform (docs, blog, changelog) and an authenticated customer portal with admin tooling. Multi-tenant: organizations own resources.

### 1.2 Users & roles

| Role | Description |
|---|---|
| Visitor | reads public content |
| Member | org member: dashboard, profile, billing view, notifications |
| Org Owner | + billing management, member management |
| Admin (platform) | cross-tenant user/org management, content management, analytics |

### 1.3 Pages & features

**PUBLIC (marketing + content):**
landing · pricing · testimonials · blog (categories, tags, authors, pagination, related) · documentation (MDX, TOC, code highlighting, interactive demos) · changelog · search · RSS · sitemap · SEO everywhere · dark mode · responsive.

**AUTH:**
register · login · logout · password reset · email verification · sessions (revocation on password change).

**CUSTOMER:**
dashboard (metrics, notifications) · profile · settings · billing (plan, invoices, upgrade) · file upload (avatar) · API keys (create/revoke).

**ADMIN:**
users · organizations · roles/permissions · product/content management · analytics overview · audit logs.

**SYSTEM:**
Actions for mutations · one REST API surface (v1) · inbound webhook (payments) · outbound email · rate limiting · observability · caching layer · testing suite · Docker/Node or platform deployment.

### 1.4 Data model (entities, not tables)
User · Session · Organization · Membership(role) · Project (tenant-owned resource) · Subscription/Invoice · Post · Doc · ChangelogEntry · Author · Category/Tag · Notification · Upload · AuditEvent · ApiKey · WebhookEvent(idempotency).

### 1.5 Non-functional requirements
- **Performance:** content pages 0 framework JS; LCP p75 < 2.0s (field); app pages < 170 KB JS gzip.
- **SEO:** all public pages indexed, canonicals/hreflang-ready, structured data on Article/Product.
- **A11y:** WCAG 2.2 AA on critical flows; axe CI gate.
- **Security:** Module 23 checklist passes; tenant isolation proven by tests; no secrets in bundles (CI scan).
- **Reliability:** health endpoint; migrations safe; webhook idempotent.
- **Testing:** pyramid per Module 30 including authz matrix.

---

## 2. YOUR DESIGN TASKS (do these before reading the review)

1. **Content architecture** — collections, schemas, references, draft workflow.
2. **Page structure** — the route tree; which files live where.
3. **Astro vs island boundaries** — every interactive widget classified, with directives + JS budget.
4. **UI framework choices** — confirm React-as-islands (or argue otherwise, in writing).
5. **Data architecture** — DB schema sketch (Drizzle), layering, tenancy columns.
6. **Authentication** — library + session model + flows list.
7. **Authorization** — permission matrix (roles × resources × actions) + isolation strategy.
8. **Actions vs endpoints** — table of every mutation/operation and its mechanism.
9. **Rendering strategy** — per-route static/SSR/deferred table (Module 14's five questions).
10. **Caching strategy** — the four cache questions per route class (Module 34).
11. **Deployment** — host(s), adapter(s), env/secrets, CI/CD pipeline.

Write each as a short document (a page each is plenty). **Commit them before reading Part 3.**

---

## 3. Reference architecture (review after your attempt)

### 3.1 Suggested folder layout — with *why*

```text
src/
  components/            # zero-JS building blocks (why: default is free)
    react/               # islands only (why: makes the JS budget visible in code review)
  layouts/               # BaseLayout + AppLayout (why: public vs portal have different chrome/policies)
  pages/                 # routing = IA; app/** and admin/** opt into SSR (why: file system as route table)
  content/               # md/mdx + data (why: content is data — Module 12)
  content.config.ts      # schemas/loaders (why: one typed source of truth)
  live.config.ts         # e.g. live pricing/changelog feeds (why: volatile data without rebuilds)
  actions/               # first-party mutations (why: typed, validated, no-JS forms)
  middleware.ts          # requestId → security headers → session context (why: one pipeline)
  server/
    auth/                # Better Auth config + flows (why: auth is a subsystem, not a page)
    db/                  # client, schema.ts, migrations (why: SQL lives in one place)
    services/            # business rules + transactions (why: authz + orchestration next to data)
    repositories/        # queries scoped by tenant (why: IDOR-proof by construction)
    permissions/         # capability checks (why: roles change; capabilities don't)
  styles/ utils/         # tokens/global css; pure helpers
```

**Folders you can delete for smaller projects:** `live.config.ts` (no volatile content), `server/repositories/` (inline in services until 3+ call sites), `components/react/` (content-only sites).

### 3.2 The master flow (already drawn in the reference file)

Request → middleware (locals) → page/action → service (authz) → repository (scoped SQL) → DTO → HTML/JSON — with cache hints along the way and islands hydrating on schedule.

### 3.3 Review rubric (grade yourself)

| Area | Good decisions | Common bad decisions |
|---|---|---|
| Islands | 2–6 per app page, lazy schedules | `client:load` on chrome; React for static |
| Content | typed collections, references | stringly-typed tags, `slug` fields ⛔ |
| Rendering | static public, SSR app, defer fragments | all-SSR "for flexibility" |
| Actions/endpoints | actions for UI, endpoints for externals | JSON fetch triples everywhere |
| Auth | cookies + library + middleware locals | JWT in localStorage; middleware-only guards |
| Authz | capabilities + scoped queries + tests | UI-hidden buttons; id-only selects |
| Data | services/repos/transactions/pooling | SQL in pages; per-request clients |
| Caching | four questions answered per class | public cache on personalized pages |
| Quality | check+tests+axe+budgets in CI | "works on my machine" |
| Deployment | pipeline with migrations+smoke | preview server as prod |

### 3.4 Final architecture review (the §96 checklist)

Run through everything you built and answer, in writing:

1. Islands & hydration — justified per directive? Budget met?
2. Server/client boundary — any `src/server` imports in islands? Any secrets in `dist/`?
3. Rendering mode per route — still correct as features grew?
4. AuthN/AuthZ — matrix tested? IDOR probes done?
5. Data — pools, transactions, indexes on tenant FKs?
6. Actions vs endpoints — consistent?
7. Middleware — fast? guard scoping right? webhook-safe?
8. Content — schema drift? draft leaks?
9. SEO/a11y/perf — numbers vs budgets?
10. Testing/observability — would you find out before users do?
11. Deployment — rollback story? migration safety?
12. Maintenance — what would a new hire misunderstand first? Fix the docs.

**Identify: good decisions · bad decisions · security issues · performance issues · scalability concerns · maintenance concerns.** That list — and the PRs that address it — is the real deliverable of the course.

---

## MENTAL MODEL — Capstone

> Architecture is **a stack of decisions you can defend**: content as data, documents as static HTML, widgets as budgeted islands, mutations as validated server functions, authorization next to the data, caches with named owners. If you can defend each layer with a sentence, you're done.

## Official Documentation
All modules' doc links apply. Start: https://docs.astro.build/en/ · decisions: Modules 14, 16, 22, 34.
