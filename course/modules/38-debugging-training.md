# Module 38 — Debugging Training

> Phase 72. Deliberately broken implementations. For each: **1) the broken code → 2) reproduce → 3) diagnose → 4) fix → 5) the mental model.** These are the real failure modes of Astro projects.

## Bug 1 — The island that never hydrates

**Broken:**
```astro
---
import Search from '../components/react/Search';
const Tag = Search;
---
<Tag client:load />
```
**Reproduce:** build; click search — nothing. Dev may also silently skip hydration.
**Diagnose:** client directives must be **statically visible to the compiler**; dynamic tags/spreads hide them.
**Fix:** `<Search client:load />` directly.
**Mental model:** directives are compile-time, not runtime props.

## Bug 2 — The wrong directive

**Broken:** `<Chart client:load />` for a 90 KB chart at page bottom.
**Reproduce:** Lighthouse INP/LCP regression; JS loads before first paint.
**Diagnose:** urgency mismatch — everything hydrates at `load`.
**Fix:** `client:visible` (+ `rootMargin` if users scroll fast).
**Mental model:** directives are schedules over a cost.

## Bug 3 — Unnecessary hydration

**Broken:** `<Nav client:load />` — static links + a menu that's just CSS.
**Reproduce:** Network tab shows React runtime on a content page.
**Fix:** Astro component + `<details>`/tiny script; 0 framework JS.
**Mental model:** interactivity ≠ framework. Escalation ladder (Module 01).

## Bug 4 — Browser-only code on the server

**Broken:** `const w = window.innerWidth` in a component rendered by `client:only`... or worse, in frontmatter.
**Reproduce:** build crash `window is not defined` (SSR path) or blank UI.
**Fix:** guard (`typeof window !== 'undefined'`), read media in an island effect, or `client:only` *if truly browser-only* (with fallback).
**Mental model:** frontmatter is server; island module scope also executes during SSR (for most frameworks).

## Bug 5 — Server-only code reaches the browser

**Broken:** island imports `src/server/db/client.ts` "for types" (which imports the driver + `process.env`).
**Reproduce:** bundle balloons; `process is not defined` in console.
**Fix:** `import type` only, or shared `types.ts` with zero runtime imports; move calls to actions.
**Mental model:** the island import graph is the client bundle.

## Bug 6 — Stale state after navigation

**Broken:** menu script binds on `DOMContentLoaded`; with `<ClientRouter />` it works once.
**Reproduce:** navigate → menu dead; full reload → works again.
**Fix:** bind init on `astro:page-load`, or `transition:persist` the widget.
**Mental model:** client navigation is a *swap*, not a new document.

## Bug 7 — Middleware bug (the wrong guard)

**Broken:** middleware redirects all unauthenticated requests → webhooks/health checks die.
**Reproduce:** Stripe webhooks 302 to `/login`.
**Fix:** path-scoped guards + webhook routes excluded (signature-authenticated).
**Mental model:** middleware runs for *every* request; policy must match route classes.

## Bug 8 — Authentication bug (cookie attributes)

**Broken:** session works on `localhost`, users bounce on prod: `SameSite=None` without `Secure`, or domain mismatch.
**Reproduce:** devtools → `Set-Cookie` rejected.
**Fix:** `HttpOnly; Secure; SameSite=Lax` + correct domain/`__Host-` policy (Module 21).
**Mental model:** cookies have a grammar; browsers enforce it.

## Bug 9 — Authorization bypass (IDOR)

**Broken:** `GET /api/orders/[id]` selects by id only.
**Reproduce:** change the UUID → another tenant's order.
**Fix:** `WHERE id = $1 AND org_id = $2` + service check (Module 22).
**Mental model:** authorization is a predicate next to the data.

## Bug 10 — Incorrect Action usage

**Broken:** form `action="/api/contact"` + `Astro.getActionResult(actions.contact)`.
**Reproduce:** page navigates to raw JSON; result UI never shows.
**Fix:** `action={actions.contact}`.
**Mental model:** actions wire both the URL *and* the result channel.

## Bug 11 — Broken form (no name attributes)

**Broken:** `<input type="email">` with no `name`.
**Reproduce:** handler receives empty `FormData`.
**Fix:** `name="email"`; remember: **no name = not submitted**.
**Mental model:** FormData is name → value.

## Bug 12 — Hydration mismatch after view transitions

**Broken:** island renders timestamp `Date.now()` on server and client.
**Reproduce:** console hydration warnings after navigation.
**Fix:** render unstable values only client-side (or `transition:persist`/`client:only` where appropriate).
**Mental model:** hydration expects server HTML ≈ first client render.

## Bug 13 — Incorrect dynamic routing

**Broken:** `[...slug].astro` `getStaticPaths` returns `params: { slug: undefined }` for the index case.
**Reproduce:** `/blog/` missing from build.
**Fix:** rest params: `slug: undefined` matches the base — verify docs for the exact behavior in your version; or use an explicit `index.astro`.
**Mental model:** params must match the filename grammar.

## Bug 14 — Cache serving stale content

**Broken:** `maxAge: 86400` on product pages; price update invisible for a day.
**Reproduce:** edit price → same HTML.
**Fix:** short `swr` + tag invalidation on writes (Module 34).
**Mental model:** every cache needs a named invalidation owner.

## Bug 15 — SSR deployment bug (standalone behind proxy)

**Broken:** standalone Node behind nginx; `Astro.url` is `http://internal:4321` — redirects/cookies break.
**Reproduce:** redirect loop on HTTPS.
**Fix:** trust/proxy headers per adapter docs (`X-Forwarded-Proto/Host`) or configure `site`/proxy correctly.
**Mental model:** the origin sees the proxy's view unless told otherwise.

## Bug 16 — Environment variable exposure

**Broken:** `PUBLIC_`-prefixed a secret "temporarily"; or `import.meta.env.API_KEY` inlined at build (v6+) into server output that got shared.
**Reproduce:** grep `dist/` for the key prefix.
**Fix:** `process.env` + rotation of the leaked key + secret scanning in CI (Module 33).
**Mental model:** bundles are published documents.

## The debugging workflow (use for anything)

1. **Reproduce** on `build`/`preview` — not just `dev`.
2. **Read the compiler/runtime** — v7 Rust compiler errors are truths.
3. **Bisect the boundary** — server vs client, build vs request, route vs middleware.
4. **Inspect the output** — View Source, `dist/`, Network tab, `curl -I`.
5. **Write the failing test** before the fix.

## Common Mistakes (debugging edition)

**BAD:** fixing the first symptom (adding `client:load` to "fix" a non-hydrating island that had a spread directive).
**GOOD:** reproduce first, name the boundary, fix the cause.
**BAD:** debugging only in `astro dev`.
**GOOD:** confirm on `build`/`preview` — half these bugs are build-only or prod-only.
**BAD:** "it works now" with no test.
**GOOD:** failing test first, fix second (Module 30).
**BAD:** reading the wrong console (client logs for server errors).
**GOOD:** boundary first: which process failed — build, server, or browser?

## Security Notes

- Several drills above (9, 15, 16) are live vulnerabilities — run them **only on local scratch apps**, never against real environments.
- When a fix touches a security-class bug, add evidence to the [security checklist](../reference/security-review-checklist.md) and a regression test.
- Debugging often means looking at real data: use seeded fixtures, and never paste session cookies or tokens into logs/issues while diagnosing.

## Performance Notes

- Debugging tools cost performance: dev overlays, verbose loggers, and sourcemaps-with-secrets stay out of production builds (Module 31).
- After perf-class fixes (Bugs 2, 3, 5, 14), re-measure against the written budget — "feels faster" isn't a metric (Module 29).

## Exercises

**Beginner.** Reproduce Bugs 1–4 in a scratch project; fix each; write a one-sentence diagnosis in your own words.

**Intermediate.** Reproduce the authz/form/cache cluster (Bugs 9, 10, 11, 14) and write a **failing test** for each before fixing.

**Production.** Turn one fix into a permanent guard: e.g. a CI bundle-grep test that fails if anyone reintroduces Bug 5 (server-only import in an island bundle).

**Architecture Challenge.** You inherit a project exhibiting **all 16 bugs**. Order the fixes by risk and by dependency, and justify the order in writing.

> **Review:** Security exposure first (16, 9, 5) → correctness (1, 6, 10, 11, 13) → reliability (7, 8, 14, 15) → performance (2, 3) → polish (4, 12). Dependencies matter: extract shared types (Bug 5) before bundle work; define cache invalidation (Bug 14) before perf tuning. Security bugs don't wait for feature work.

**Debugging Challenge.** The capstone drill: take your own design from the [capstone PRD](../projects/capstone-prd-and-architecture.md), plant three bugs from this module in a throwaway branch, and trade with a study partner. Diagnose theirs in under 30 minutes using the workflow — the goal is the speed of naming the right boundary, not the fix itself.

## MENTAL MODEL — Debugging

> Every bug above is a **boundary confusion**: compile-time vs runtime, server vs browser, request vs build, cache vs origin, mine vs tenant's. Debug by naming which boundary you're on — then look at the artifact on the correct side (HTML, bundle, header, SQL).

## Official Documentation

- Troubleshooting: https://docs.astro.build/en/guides/troubleshooting/
- Common errors: https://docs.astro.build/en/reference/error-reference/
- Upgrade guides: https://docs.astro.build/en/upgrade-astro/

## What I Should Know Before Continuing

1. Reproduce Bug 1 and Bug 5 yourself — then explain them to a rubber duck in one sentence each.
2. Which bugs disappear in `dev` but appear in `build`? Why?
3. What's your personal debugging order for "works locally, broken in prod"?
4. Which boundary does IDOR violate? Which does the hydration mismatch violate?
