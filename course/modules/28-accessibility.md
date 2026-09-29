# Module 28 — Accessibility

> Phase 23/44. Less JavaScript *often* makes accessibility easier — but not automatically. Semantic HTML is a skill; this module is the checklist + patterns.

## Concept: accessibility is document quality

Astro's default (server HTML, native controls) is a **strong a11y baseline** — real links, real forms, real headings survive. Islands can regress it (div-buttons, focus traps, aria-less widgets). Hence: **HTML first, then enhance without losing semantics.**

## The core checklist

### 1. Landmarks & headings
```html
<header> <nav aria-label="Main"> <main id="main"> <aside> <footer>
<h1> → <h2> → <h3>   <!-- no skipped levels, one h1 per page -->
<a class="skip-link" href="#main">Skip to content</a>
```

### 2. Keyboard
- Everything interactive is focusable and operable (Tab, Enter, Space, arrows where expected).
- Visible focus: `:focus-visible` styles — never `outline: none` without replacement.
- No keyboard traps (modals must close on Esc and restore focus).

### 3. Forms (Module 17's a11y block, expanded)
- Label **every** input; group with `fieldset`/`legend`.
- Errors: `role="alert"` + `aria-describedby` links; never color-only signaling.
- Pending: `aria-busy`; success: `role="status"`.

### 4. ARIA rules
- **First rule of ARIA: don't use ARIA** if a native element works (`<button>`, `<details>`, `<dialog>`, `<nav>`).
- ARIA describes state the DOM can't express (`aria-expanded`, `aria-selected`); it doesn't create keyboard behavior — JS does.
- Test patterns against APG ([WAI-ARIA Authoring Practices](https://www.w3.org/WAI/ARIA/apg/)).

### 5. Media & motion
- Images: meaningful `alt`, `alt=""` decorative (Module 25).
- Video: captions/transcripts.
- `prefers-reduced-motion` honored (Module 27).
- Animations never required to understand content.

### 6. Contrast
- Text ≥ 4.5:1 (3:1 large text); UI components ≥ 3:1. Test dark mode too.

### 7. Screen reader experience
- Meaningful link text ("Read the pricing guide", not "click here").
- Live regions for async updates (islands!).
- Language attribute (`<html lang>`), `hreflang` on translations (Module 35).

## The progressive enhancement → a11y link

| Feature | Baseline (no JS) | Enhanced | A11y preserved because |
|---|---|---|---|
| Newsletter | form POST | inline errors | server-rendered `role="alert"` |
| Search | GET form results | instant island | results still a real list |
| Tabs | all panels visible | tab UI | ARIA tab pattern + keyboard model |
| Nav menu | `<details>` or links | animated drawer | focus managed on toggle/close |
| Dialog | `<dialog>` + `:target` fallback | `showModal()` | native focus handling |

**The pattern:** baseline is *usable and semantic*; the island/script **adds** behavior without removing the semantics.

## Testing

- Keyboard-only pass on every interactive feature.
- Screen reader spot-checks (VoiceOver/NVDA) on critical flows (login, checkout).
- axe-core in CI (Playwright + axe — Module 30).
- Lighthouse a11y as a smoke signal, not the verdict.

## Common Mistakes

**BAD:** `<div onClick>` menu items.

**GOOD:** `<button aria-expanded>` + keyboard handlers per APG.

**BAD:** `aria-label` on everything "just in case."

**GOOD:** labels where semantics need names; nothing where they don't.

**BAD:** modal island with no focus trap and no Esc.

**GOOD:** `<dialog>` or tested focus management.

**BAD:** color-only error states; 2px fonts; 100%-width carousels that trap scroll.

**GOOD:** contrast-checked tokens (Module 06), tested zoom to 200%.

## Security Notes

- Accessible error messages must not leak ("No user with that email" vs "Invalid credentials" — Module 21).
- `aria-live` announcements from server errors: escape content (they're normal DOM text — fine).

## Performance Notes

- A11y and perf share enemies: hydration-heavy widgets, motion libraries, DOM churn.
- Skip links and focus styles are ~free.

## Exercises

**Beginner.** Audit a page: headings order, alt text, labels, focus styles. Fix 10 issues.

**Intermediate.** Accessible accordion (`<details>` baseline + optional animation) and an accessible modal (`<dialog>`) — keyboard-tested.

**Production.** Playwright + axe test suite on 5 critical routes; zero serious/critical violations; wire into CI.

**Architecture Challenge.** Data table island with sorting, filtering, pagination — a11y plan before code.

> **Review:** sortable headers = `<button>` in `<th>` with `aria-sort`; filter region = form (GET baseline); pagination = real links; announce result counts via `role="status"`; keyboard row navigation if rows are interactive.

**Debugging Challenge.** Screen reader announces nothing when island search results update. Fix: `role="status"`/`aria-live="polite"` region announcing result count.

## MENTAL MODEL — Accessibility

> Accessible = **operable by keyboard, describable by assistive tech, and honest without JS**. Start from semantic HTML; every enhancement must carry its keyboard model and announcements with it. If a widget needs 5 ARIA attributes to seem what it is, maybe it should just *be* the native element.

## Official Documentation

- Astro accessibility: https://docs.astro.build/en/guides/accessibility/
- WAI-ARIA APG: https://www.w3.org/WAI/ARIA/apg/
- WCAG quickref: https://www.w3.org/WAI/WCAG22/quickref/

## What I Should Know Before Continuing

1. First rule of ARIA. Where do you look up a widget's keyboard model?
2. The a11y states for form error/pending/success — roles and wiring.
3. How does progressive enhancement specifically protect accessibility?
4. What can axe in CI catch — and what must a human test?
