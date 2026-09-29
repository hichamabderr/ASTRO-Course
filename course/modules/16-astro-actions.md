# Module 16 — Astro Actions (Deep Dive)

> Phase 14. **Why Actions exist:** calling your own server from your own UI should be **type-safe end to end** — input validated, errors typed, calls generated — instead of hand-rolled fetch + endpoint + validation triples.

## Concept

An **Action** is a named, validated server function defined in `src/actions/index.ts` and callable from HTML forms and client scripts with full TypeScript types flowing to the caller.

```text
Form / island  ──call──►  Action  ──validate (Zod)──►  handler(input, context)
                                             │              │
                                             ▼              ▼
                                        input errors    ActionError / data
                                        (typed)         (serialized to caller)
```

## Defining actions (verified API)

```ts
// src/actions/index.ts
import { defineAction, ActionError } from 'astro:actions';
import { z } from 'astro/zod';

export const server = {
  newsletter: defineAction({
    accept: 'form',
    input: z.object({
      email: z.email(),
      terms: z.boolean(),
    }),
    handler: async ({ email, terms }) => {
      if (!terms) {
        throw new ActionError({ code: 'BAD_REQUEST', message: 'You must accept the terms.' });
      }
      await subscribe(email);
      return { ok: true };
    },
  }),

  createComment: defineAction({
    accept: 'form',
    input: z.object({
      postId: z.string(),
      body: z.string().min(1).max(2000),
    }),
    handler: async (input, ctx) => {
      const user = ctx.locals.user;
      if (!user) throw new ActionError({ code: 'UNAUTHORIZED', message: 'Log in to comment.' });
      return await createComment({ ...input, authorId: user.id });
    },
  }),

  getOrder: defineAction({
    input: z.object({ id: z.string() }),     // default accept: 'json'
    handler: async ({ id }, ctx) => {
      const order = await ctx.locals.services.orders.getForUser(id, ctx.locals.user!.id);
      if (!order) throw new ActionError({ code: 'NOT_FOUND', message: 'No such order.' });
      return order;
    },
  }),
};
```

**Key verified facts:**
- `defineAction({ accept, input, handler })` — `accept: 'form' | 'json'` (default `json`).
- `handler(input, context)` — `context` is a subset of endpoint context (`locals`, `cookies`, `request`, …).
- Invalid input ⇒ `BAD_REQUEST` **without calling the handler**.
- Form validators map: text → `z.string()`, numbers → `z.number()`, checkboxes → `z.boolean()` (coercion variants documented in the [Actions guide](https://docs.astro.build/en/guides/actions/)), files → `z.instanceof(File)`, repeated names → `z.array(...)`.
- `ActionError` codes map to HTTP status (`UNAUTHORIZED`, `NOT_FOUND`, `FORBIDDEN`, …).
- Organize: `server: { user: userActions, billing: billingActions }` → `actions.user.*`.

## Calling from an HTML form (progressive enhancement)

```astro
---
import { actions } from 'astro:actions';
const result = Astro.getActionResult(actions.newsletter);
---

<form method="POST" action={actions.newsletter}>
  <label>Email <input type="email" name="email" required /></label>
  <label><input type="checkbox" name="terms" /> I accept</label>
  <button>Subscribe</button>
  {result?.error && <p role="alert">{result.error.message}</p>}
  {result?.data && <p role="status">Subscribed!</p>}
</form>
```

**Without JS:** the form POSTs to the action URL and re-renders the page — full server round trip, works everywhere. **Redirect on success** with `onSubmit` handlers or return redirects from the action path pattern documented in the Actions guide (`redirect()` on success in handler context).

## Calling from a client script / island

```ts
// vanilla <script> or island code
import { actions, isInputError } from 'astro:actions';

const { data, error } = await actions.newsletter({ email, terms: true });
// data/error are TYPED. No hand-written fetch contract.

if (isInputError(error)) {
  console.log(error.fields.email); // per-field messages
}
```

Safe-style calls also exist (`actions.newsletter.safe(...)`) returning `{ data, error }` without throwing — check the [Actions API reference](https://docs.astro.build/en/reference/modules/astro-actions/) for current surface (`getActionContext`, `getActionPath`, `ACTION_QUERY_PARAMS` are also exported for advanced cases).

## Action vs Endpoint vs HTML form vs client fetch

| | Action | Endpoint | Native form | Client fetch |
|---|---|---|---|---|
| Typed calls | ✅ generated | ❌ hand contract | n/a | ❌ |
| Works without JS | ✅ (`accept:'form'`) | ❌ | ✅ | ❌ |
| For external consumers | awkward | ✅ | — | — |
| Validation | Zod built-in | manual Zod | manual | browser only |
| Best for | first-party UI mutations | public APIs/webhooks | baseline UX | islands needing async |

## Real-world action set (capstone preview)

```text
subscribeToNewsletter()   accept:'form'  — public, rate-limited
createComment()           accept:'form'  — authz: comment on published post only
updateProfile()           accept:'form'  — auth + field-level validation
createOrder()             json           — auth + transaction + stock check
uploadAvatar()            accept:'form'  — z.instanceof(File) + Module 36 rules
```

## Middleware integration

Actions run **inside the middleware pipeline** — `context.locals` from `src/middleware.ts` is populated. That's how `ctx.locals.user` exists above. But **authorization must still be enforced inside the handler** (Module 22): middleware is convenience; the handler is the gate.

## Common Mistakes

**BAD:** `handler` trusting `input.postId` — user A updates user B's post (IDOR).

**GOOD:** `orders.getForUser(id, userId)` — ownership in the query.

**BAD:** returning entities with internal fields (`passwordHash`, `stripeCustomerId`).

**GOOD:** explicit DTO mapping in the action.

**BAD:** 15 actions doing "parse request.json() manually" instead of one typed action.

**GOOD:** that's what actions replace; endpoints remain for external consumers.

**BAD:** catching all errors and returning `{ ok: false }` with 200.

**GOOD:** `ActionError` for expected failures (typed!), let unexpected errors bubble to middleware error handling (500 + logged request ID).

**BAD:** putting long-running jobs (video encode) inline.

**GOOD:** enqueue + return job id; show progress (island polls or SSE).

## Security Notes

- **CSRF:** actions invoked from forms on your origin; still enforce `Origin`/`Sec-Fetch-Site` checks in middleware for state-changing calls, and use `SameSite=Lax|Strict` session cookies (Module 23).
- **AuthZ per action, per resource** — never rely on "the middleware said user is logged in."
- Rate-limit by session/IP at middleware or reverse proxy.
- `getActionPath`/query-param invocation: treat as public HTTP surface (it is) — same protections as endpoints.

## Performance Notes

- Actions are lazy: no JS until an island calls them. The form path ships 0 JS.
- Keep handlers thin: validation → service → repository (Module 19). Heavy work → queues.
- Payload size: return DTOs, not giant joined rows.

## Exercises

**Beginner.** `subscribeToNewsletter` with `accept: 'form'`; works with JS disabled; shows typed field errors with JS enabled (small `<script>` using `isInputError`).

**Intermediate.** `createComment` + `deleteComment` with ownership checks and `ActionError` codes; test via curl to the action path.

**Production.** `createOrder` with transactional stock decrement (Module 19) + idempotency key header; return typed order DTO; integrate request-ID logging.

**Architecture Challenge.** A checkout flow: cart mutation, coupon apply, payment intent, order create. Which are actions, which are endpoints (Stripe webhook!), where does idempotency live?

> **Review:** `updateCart`, `applyCoupon`, `createOrder` = actions (first-party, typed, authz inside); Stripe `webhook` = endpoint (signature verification, Module 36); payment intent creation = action calling Stripe SDK server-side; idempotency key generated client-side, stored server-side on the order.

**Debugging Challenge.** Form POSTs and returns JSON error `BAD_REQUEST` in the browser instead of re-rendering the page. Cause: `action={actions.newsletter}` missing — you posted to a hand-written `/api/subscribe` while `getActionResult` expects the action's path. Fix: use `action={actions.x}` in the form (Astro wires the correct URL + result flow).

## MENTAL MODEL — Actions

> Actions are **RPC with receipts**: one definition produces validation, typing, error shape, HTTP plumbing, and a no-JS form target. They exist to make the *right* thing (server-validated mutations) the *easy* thing. The handler is still a public HTTP surface — authorize inside it.

## Official Documentation

- Actions guide: https://docs.astro.build/en/guides/actions/
- Actions API: https://docs.astro.build/en/reference/modules/astro-actions/
- Middleware: https://docs.astro.build/en/guides/middleware/

## What I Should Know Before Continuing

1. Anatomy of `defineAction` — `accept`, `input`, `handler`, error types.
2. How does the no-JS form path work?
3. Action vs endpoint decision rule.
4. Why does authz live inside the handler even with middleware?
