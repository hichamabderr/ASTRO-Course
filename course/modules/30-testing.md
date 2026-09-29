# Module 30 — Testing

> Phases 25/60/87. A complete testing strategy: the pyramid, what to test where, and the current tool stack (Vitest, Testing Library, Playwright, axe).

## The test pyramid for Astro

```text
                E2E (Playwright)          flows: login, checkout, publish
             Integration (Vitest)         actions, endpoints, middleware, services
          Component (Testing Library)     island behavior + a11y queries
              Unit (Vitest)               pure logic: permissions, schema, utils
```

**What lives where — Astro-specific:**

| Code | Test level | Tool |
|---|---|---|
| Zod schemas, permissions, utils | unit | Vitest |
| Services/repositories | integration (test DB) | Vitest + Drizzle + ephemeral Postgres |
| Actions/endpoints/middleware | integration (call handler with mocked context) | Vitest |
| `.astro` components (static HTML) | Container API render tests | Vitest + `astro/container` |
| React islands | component tests | Testing Library + jsdom |
| Critical user flows | E2E | Playwright (+ axe) |
| Accessibility | component + E2E | axe-core, keyboard tests |

## Unit + integration (Vitest)

```ts
// src/server/permissions/permissions.test.ts
import { expect, test } from 'vitest';
import { can } from './index';

test('members cannot manage billing', () => {
  expect(can({ role: 'member' }, 'billing')).toBe(false);
});

// Action handler test (integration): pass a fake APIContext
test('createComment requires auth', async () => {
  const { server } = await import('../../actions');
  // invoke handler with locals.user = null → expect ActionError UNAUTHORIZED
});
```

## Component tests

**Islands (React):** Testing Library with user-event — assert roles/labels, keyboard flows.

**Astro components:** the **Container API** (`astro/container`) renders `.astro` to HTML in tests. Note v6/v7 changes: components can't render in Vitest *client* environments (use the container in node env), and `getContainerRenderer()` now imports from the integration's `container-renderer` entrypoint (e.g. `@astrojs/react/container-renderer`).

```ts
import { experimental_AstroContainer as AstroContainer } from 'astro/container';
import Card from '../components/Card.astro';

const container = await AstroContainer();
const html = await container.renderToString(Card, { props: { title: 'Hi' } });
expect(html).toContain('Hi');
```

(Verify the exact container import against the current [testing guide](https://docs.astro.build/en/guides/testing/) — it evolves with minors.)

## E2E (Playwright)

```ts
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test('login → dashboard shows user name', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('ada@example.com');
  await page.getByLabel('Password').fill(process.env.TEST_PW!);
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page.getByRole('heading', { name: /Hi, Ada/ })).toBeVisible();
});
```

Run against `astro preview` or the deployed preview URL. Seed test data; never run against prod DBs.

## Accessibility testing

- `@axe-core/playwright` scans in E2E; fail CI on serious/critical.
- Manual: keyboard-only runs + screen reader spot checks (Module 28) — automation catches ~30–40% of issues.

## What NOT to test

- Third-party library internals.
- Snapshotting entire pages (brittle) — snapshot small HTML fragments if at all.
- The same assertion at three levels (pick the lowest level that catches the regression).

## Common Mistakes

**BAD:** only E2E tests; suite takes 40 minutes; nobody runs it.

**GOOD:** pyramid — most logic tested in milliseconds at unit/integration.

**BAD:** testing implementation ("component has class .btn-primary").

**GOOD:** behavior ("button with accessible name 'Save' submits").

**BAD:** no tests around authz — the IDOR ships.

**GOOD:** matrix tests: anonymous/member/other-tenant/admin × route → 401/403/404/200 (Modules 21–22).

**BAD:** unit-testing Zod schemas' own validation.

**GOOD:** test *your* constraints (the exact rules you wrote).

## Security Notes

- Test that secrets don't appear in HTML/JS output (grep `dist/` in CI for known secret prefixes).
- E2E credentials: env-only, test accounts, rotated.

## Performance Notes

- Parallelize Playwright; run smoke E2E per PR, full suite nightly.
- Container tests are fast; keep most coverage there.

## Exercises

**Beginner.** Vitest unit tests for a permissions helper + a Zod schema refinement.

**Intermediate.** Playwright test for newsletter form with JS disabled (form POST path!) and enabled (action path).

**Production.** Authz matrix test for one action; axe scan in CI blocking merges on serious violations.

**Architecture Challenge.** Legacy app has 900 slow Selenium tests. Migration plan to this pyramid?

> **Review:** quarantine + smoke-port the top 20 business flows to Playwright; move logic assertions down to Vitest as code is touched; delete UI-implementation tests; add axe. Gate by risk, not nostalgia.

**Debugging Challenge.** Container test fails to render a component that imports a React island with `client:load`. Container doesn't hydrate — it renders; client directives are no-ops/unsupported in this context. Fix: test the island separately with Testing Library; container-test the static wrapper.

## MENTAL MODEL — Testing

> Test **contracts** at the **cheapest level that can catch their violation**: logic in unit tests, server behavior in integration tests, user journeys in E2E, accessibility in both automated and human passes. Authorization deserves a *matrix*, not a hope.

## Official Documentation

- Testing guide: https://docs.astro.build/en/guides/testing/
- Container API: https://docs.astro.build/en/reference/container-reference/
- Playwright: https://playwright.dev/ · Vitest: https://vitest.dev/ · Testing Library: https://testing-library.com/

## What I Should Know Before Continuing

1. Draw the pyramid; place actions, islands, permissions, checkout flow.
2. How do you test an `.astro` component? An island? Why differently?
3. What belongs in the authz matrix test?
4. Why can't the container API replace Testing Library for islands?
