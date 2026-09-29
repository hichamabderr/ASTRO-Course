# Module 17 — Forms & Progressive Enhancement

> Phases 22 & 86. The native `<form>` is the web's oldest API for "send this data to the server" — and it still beats most JS form stacks for the 90% case.

## The escalation ladder for forms

```text
1. Native <form method="POST"> → server processes → redirect/render     (works everywhere)
2. + HTML validation attributes (required, type=email, pattern, min/max)
3. + CSS state styling (:invalid, :user-invalid, :focus-visible)
4. + tiny <script> for inline errors / optimistic UI                    (progressive enhancement)
5. + Action for typed server processing                                 (recommended server target)
6. Full React form island                                               (only for genuinely complex UI)
```

## Native form semantics (get these right)

```astro
<form method="POST" action={actions.createComment} enctype="multipart/form-data">
  <fieldset>
    <legend>Leave a comment</legend>

    <label for="body">Comment</label>
    <textarea id="body" name="body" required maxlength="2000"></textarea>

    <label>
      <input type="checkbox" name="notify" value="yes" /> Notify me
    </label>

    <button type="submit">Post</button>
  </fieldset>
</form>
```

Rules: **every input has a name**; labels are bound (`for`/`id` or wrapping); `fieldset`/`legend` for groups; buttons: default `type="submit"` — set `type="button"` otherwise!

## Server processing (the Action path)

```astro
---
import { actions } from 'astro:actions';
const result = Astro.getActionResult(actions.createComment);
---

<form method="POST" action={actions.createComment}>
  <textarea name="body" required></textarea>
  {result?.error && (
    <ul role="alert">
      {Object.entries(result.error.fields ?? {}).map(([k, v]) => <li>{k}: {v}</li>)}
    </ul>
  )}
  <button>Post</button>
</form>
{result?.data && <p role="status">Comment posted (#{result.data.id})</p>}
```

**The flow without JS:** POST → Zod validation → handler → re-render with `getActionResult`. Browsers show native constraint errors before any request when you use HTML validation attributes.

## The enhanced island (rung 4–6)

When UX demands inline validation as-you-type, multi-step wizards, or dependent fields:

```tsx
// src/components/react/CheckoutForm.tsx  (island: client:load)
import { actions, isInputError } from 'astro:actions';
import { useState } from 'react';

export function CheckoutForm() {
  const [errors, setErrors] = useState<Record<string, string>>({});
  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    const fd = new FormData(e.currentTarget);
    const { data, error } = await actions.createOrder(Object.fromEntries(fd));
    if (isInputError(error)) setErrors(error.fields as Record<string, string>);
    else if (data) location.href = `/orders/${data.id}`;   // real navigation
  }
  return (
    <form onSubmit={onSubmit} noValidate={false} /* keep native attrs as baseline! */>
      {/* fields mirror the server action's input schema */}
    </form>
  );
}
```

Keep **the same action** as the server target: with JS, enhanced; without JS, the form can still `POST` to `actions.createOrder` (provide `action={actions.createOrder}` and skip `preventDefault` on failure paths — the pattern in the official Actions docs).

## Pending states & errors (a11y-first)

- Pending: `aria-busy="true"` on the form; disable the submit button; `role="status"` region for "Saving…".
- Errors: `role="alert"` list linked to fields via `aria-describedby`.
- Success: `role="status"` (polite) — or a redirect (PRG pattern) to avoid resubmits.

## Progressive enhancement, generally

> **The page should work without JavaScript where possible. Then enhance it.**

| Feature | HTML fallback | Enhancement |
|---|---|---|
| Newsletter | form POST | inline validation + toast |
| Search | GET form → results page | island filters instantly (URL still updates) |
| Login | form POST | inline errors, optimistic redirect |
| Comments | form POST | optimistic append |
| Navigation | real links | ClientRouter transitions (Module 27) |
| Tabs | all panels visible | tab JS with correct ARIA |

**HTML-first vs SPA-first:** SPA-first puts the burden on the boot bundle and the back button. HTML-first puts it on the server — which Astro is built for. Enhancement must never *remove* baseline capability.

## Common Mistakes

**BAD:** `<div onClick>` submit "buttons" (keyboard/screen-reader death).

**GOOD:** `<button type="submit">`.

**BAD:** client-side-only validation.

**GOOD:** HTML attrs + Zod on the server; client checks are UX, not security.

**BAD:** losing form state on error re-render.

**GOOD:** repopulate from submitted values (or keep the island state).

**BAD:** `noValidate` everywhere because a JS library owns the form.

**GOOD:** native constraints first; `noValidate` only when you *fully* replace messaging with accessible equivalents.

**BAD:** giant React form for a 3-field contact form.

**GOOD:** native form + action; maybe a 2 KB script.

## Security Notes

- Server validation is the only validation that counts (Module 23).
- CSRF: `SameSite` cookies + origin checks in middleware; actions/POST endpoints must be protected.
- File inputs: `accept` attribute is advisory — validate server-side (Module 36).

## Performance Notes

- Native forms ship 0 JS and cost 0 hydration. This is the cheapest UX on the web.
- Islands for forms cost framework + state logic — justify with interaction complexity.

## Exercises

**Beginner.** Contact form → Action with Zod; works with JS off; native `required`/`type="email"`.

**Intermediate.** Add inline per-field errors with a `<script>` + `isInputError`; preserve values on failure; `aria-describedby` wiring.

**Production.** Multi-step signup (account → profile → plan) with URL-driven step state (`?step=2`), full no-JS fallback (server renders each step), pending + error a11y.

**Architecture Challenge.** "We need a form builder with drag-and-drop fields." Native forms? Compare with: *the *builder* is a client island (real component state); the *produced forms* are native HTML + actions; submissions stay progressive.*

**Debugging Challenge.** Form always re-renders full page even with JS enabled and shows nothing on success. You used `action="/api/contact"` (endpoint returning JSON) instead of `action={actions.contact}`; the browser navigated to the JSON. Fix with the action path and `getActionResult`.

## MENTAL MODEL — Forms

> A form is a **protocol**: names, types, constraints, and one URL that speaks POST. Enhance the *experience* (inline errors, pending states) but never the *contract* — the server must remain correct with zero client code.

## Official Documentation

- Actions (form usage): https://docs.astro.build/en/guides/actions/#accepting-form-data-from-an-action
- Recipes (forms): https://docs.astro.build/en/recipes/build-forms/
- HTML forms (MDN): https://developer.mozilla.org/en-US/docs/Web/HTML/Element/form

## What I Should Know Before Continuing

1. The six rungs of the form ladder. Where does your contact form sit? Your checkout?
2. How do you re-render with errors without JS?
3. Which a11y roles belong to errors, pending, and success states?
4. Why is client validation never the security boundary?
