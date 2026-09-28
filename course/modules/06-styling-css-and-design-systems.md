# Module 06 — Styling, CSS Architecture & Design Systems

> Phase 5. Scoped CSS, global tokens, Tailwind 4 in an Astro world, and the honest cost of React component libraries.

## Concept: CSS is the styling platform; tools are optional accelerators

Astro ships first-class CSS handling:

| Mechanism | What it does |
|---|---|
| `<style>` in `.astro` | **scoped** to the component (attribute selectors) |
| `<style is:global>` | global rules (use sparingly) |
| `src/styles/global.css` imported in layout | resets, tokens, typography |
| `define:vars` on `<style>` | pass server values as CSS custom properties |
| CSS `@import`, nesting, `@layer`, container queries | native, processed by Vite |

## Design tokens with CSS variables

```css
/* src/styles/global.css */
:root {
  --color-bg: #fff;
  --color-fg: #111;
  --color-accent: #5b21b6;
  --space-1: 0.25rem;
  --radius-md: 0.5rem;
  --font-sans: system-ui, sans-serif;
}
.dark {
  --color-bg: #111;
  --color-fg: #eee;
}
@media (prefers-color-scheme: dark) {
  :root:not(.light) { /* same overrides */ }
}
```

Dark mode is a **token swap**, not a component concern. Toggle via `class="dark"` on `<html>` (persisted choice → tiny script; system preference → media query).

## Modern CSS you should use instead of JS

| Need | Old way | Platform way |
|---|---|---|
| Accordion | React state | `<details>`/`<summary>` |
| Modal | JS library | `<dialog>` + `showModal()` (or pure CSS `:target` patterns) |
| Menu | JS library | `<details>` or `popover` attribute |
| Tabs | JS | radio inputs + `:checked` or `:has()` (with care for a11y) |
| Animation on scroll | JS observer | CSS scroll-driven animations / `IntersectionObserver` in tiny script |
| Responsive components | JS resize | **container queries** |
| Conditional layout | JS | `:has()` |

Fewer moving parts, better accessibility defaults, zero hydration.

## Tailwind CSS 4 (the correct modern setup)

**OLD vs MODERN:** the old `@astrojs/tailwind` integration targeted Tailwind v3. **Tailwind v4 uses the `@tailwindcss/vite` plugin.**

```bash
npm install tailwindcss @tailwindcss/vite
```

```js
// astro.config.mjs
import { defineConfig } from 'astro/config';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
  vite: { plugins: [tailwindcss()] },
});
```

```css
/* src/styles/global.css */
@import "tailwindcss";

@theme {
  --color-brand: #5b21b6;
  --font-display: "Inter", sans-serif;
}
```

Tailwind v4 is CSS-first (`@theme` tokens, no `tailwind.config.js` required). Rules for Astro:

1. Utility classes are fine **in components** — but keep long class lists behind `class:list` compositions or Astro components (`<Button>` as an Astro component).
2. Never wrap static content in React "to reuse my Tailwind React Button."
3. Ship purged CSS only — Tailwind's Vite plugin handles scanning `.astro` files.

## When plain CSS is preferable

- Content documents (prose styling: `src/styles/prose.css`)
- Tiny sites (< 10 components)
- Design systems where cascade/layers give cleaner theming than utility soup
- Teams fluent in CSS but not Tailwind

## Component libraries: the honest cost

**shadcn/ui** and similar React kits are excellent **inside React islands** (Module 09) — dashboards, complex forms. They are a **disaster** when dragged onto static pages: every styled button becomes a hydrated React node.

**BAD:** the whole marketing site styled with a React component library inside `client:load` wrappers.
**GOOD:** Astro components (or Tailwind utilities) for documents; shadcn inside the dashboard island only.

## CSS architecture patterns

```text
src/styles/
  tokens.css     # custom properties only
  reset.css      # modern reset
  prose.css      # Markdown content styling
  utilities.css  # few shared utilities (.visually-hidden, .container)
```

Use `@layer components` when specificity wars start. Prefer composition over deep selectors.

## Common Mistakes

**BAD:** `!important` wars between layout and component styles.

**GOOD:** tokens + layers + scoping; fix the cascade instead of escalating.

**BAD:** inline `style={`color: ${userColor}`}` from user input (style injection).

**GOOD:** map user choices to allow-listed CSS variables/classes.

**BAD:** two theme systems (Tailwind `dark:` + custom `.dark` variables) fighting.

**GOOD:** one theme mechanism; if using Tailwind 4, express dark mode via `@theme`/CSS vars.

## Security Notes

- Never interpolate untrusted strings into `style` attributes or `define:vars` without allow-listing (CSS injection → data exfil via `url()`).
- Fonts from third parties = extra RTTs + privacy implications; self-host when possible (Module 29).

## Performance Notes

- CSS is render-blocking: keep global CSS small; component CSS is code-split per page automatically.
- Container queries beat JS resize listeners for responsiveness.
- Animation: prefer `transform`/`opacity`; respect `prefers-reduced-motion` (Module 28/27).

## Exercises

**Beginner.** Global tokens + `Button.astro` with `class:list` variants (`primary`, `ghost`), scoped styles only.

**Intermediate.** Dark mode toggle: `data-theme` on `<html>`, persisted in `localStorage`, system preference fallback, no flash (inline head script).

**Production.** Port one page to Tailwind 4 via `@tailwindcss/vite`; measure CSS payload before/after.

**Architecture Challenge.** Design system with 40 components, 3 frameworks (React islands + Vue legacy widget). How do you style? Compare with: *tokens + CSS variables as the single source of truth; framework components consume the same vars; per-framework kits only where the framework component is unavoidable; never "unify" by hydrating everything.*

**Debugging Challenge.** Scoped styles don't apply to a child component's root element (classic scoping surprise). Fix with inherited custom properties or `:global` on a wrapper — explain why scoping attributes live on the *authoring* component's elements only.

## MENTAL MODEL — Styling

> Style in the cascade, not in the component tree. Tokens at the root, scoped rules per component, platforms features (details, dialog, popover, container queries) before JavaScript, and UI kits only inside islands that already justify their existence.

## Official Documentation

- Styling: https://docs.astro.build/en/guides/styling/
- Tailwind: https://docs.astro.build/en/guides/styling/#tailwind (integration directory for v4 setup)
- Scoped styles reference: https://docs.astro.build/en/reference/directives-reference/#style-directives

## What I Should Know Before Continuing

1. How does `<style>` scoping actually work? When does it surprise you?
2. Correct Tailwind 4 installation for Astro 7 — and the old way you must not copy.
3. Name three interactions CSS/platform handles that you'd previously have written JS for.
4. Why is "shadcn everywhere" an architecture smell in Astro?
