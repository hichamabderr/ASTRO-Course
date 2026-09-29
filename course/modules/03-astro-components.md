# Module 03 — Astro Components

> Phase 3. The `.astro` component format: frontmatter, expressions, scoped styles, scripts — and exactly when/where each part executes.

## Concept: one file, two halves

```astro
---
// 1. FRONTMATTER (server-only TypeScript)
import Card from './Card.astro';
interface Props { title: string }
const { title } = Astro.props;
const data = await fetchSomething(); // runs at build or request time
---

<!-- 2. TEMPLATE (compiles to HTML) -->
<h1 class="title">{title}</h1>
<Card>Slot content</Card>

<style>
  .title { color: var(--color-accent); } /* scoped to this component */
</style>

<script>
  // 3. SCRIPT (browser, bundled by Vite) — runs AFTER hydration of the document
  console.log('progressive enhancement lives here');
</script>
```

**Execution timeline:**

```text
build/request:  frontmatter → template expressions → HTML string + scoped CSS
browser:        paint HTML → run <script> (module, deferred) → islands hydrate per directive
```

## Template language in 60 seconds

Astro's template is **HTML with JSX-like expressions**, not JSX:

```astro
---
const user = { name: 'Ada', admin: true };
const items = ['HTML', 'CSS', 'Astro'];
---

<h1>Hello {user.name}</h1>
{user.admin && <p class="badge">Admin</p>}
<ul>
  {items.map((item) => <li>{item}</li>)}
</ul>
<p class:list={['base', { highlight: user.admin }]} />
```

- `{expr}` renders and **HTML-escapes** strings.
- `class:list` / `set:html` / `set:text` are template directives (see Module 08 reference).
- **v7 note (Rust compiler):** unclosed tags are now **errors**; invalid nesting (e.g. `<div>` inside `<p>`) is no longer silently fixed. Write valid HTML.

## Component behaviors that surprise React developers

| Behavior | Astro |
|---|---|
| State | **None at runtime.** Frontmatter variables are per-render. |
| Rerender | Doesn't exist. Rebuild or new request. |
| Lifecycle | No `useEffect`. Use frontmatter `await` for data. |
| Client JS | Only in `<script>` tags or `client:*` islands |
| Children | `<slot />` (not `props.children`) |
| CSS | scoped by default (class + `data-astro-cid-*`) |

## Scoped styles & global styles

```astro
<style>
  /* scoped: compiled to .card[data-astro-cid-xxxx] */
  .card { border: 1px solid #ddd; }
  :global(.dark) .card { border-color: #333; } /* escape hatch, rare */
</style>

<style is:global>
  /* entire app reset — better in src/styles/global.css imported by the layout */
</style>
```

Specificity, cascade layers, and design tokens: Module 06.

## Scripts: the progressive-enhancement layer

```astro
---
// server: decide WHAT the markup needs
const navId = 'site-nav';
---

<nav id={navId}>…</nav>

<script>
  // browser: enhance the existing DOM. Bundled + TS + deduped across pages.
  const nav = document.getElementById('site-nav')!;
  nav.querySelector('button')!.addEventListener('click', () => nav.classList.toggle('open'));
</script>
```

Rules of thumb:
- `<script>` is processed: TypeScript, npm imports, minified, **deduplicated** even if the component renders multiple times.
- `is:inline` opts out (raw, per-instance, no imports) — for third-party snippets or `define:vars` only.
- Prefer **event delegation + tiny scripts** over framework islands for simple UI.
- With `<ClientRouter />` (Module 27), `astro:page-load` is the safe init event instead of `DOMContentLoaded`.

## Astro component vs island component — the rule

| | `.astro` component | React/Vue/Svelte component |
|---|---|---|
| Needs interactivity? | no | **only then** |
| Ships JS | 0 | framework + component |
| Runs | server/build | server (optional SSR) + browser |
| Cost | ~0 | per-island budget |

## Common Mistakes

**BAD:** `fetch()` to your own API route inside frontmatter of a prerendered page at request-time patterns.

**GOOD:** call the service/db directly in frontmatter (build: at build; SSR: at request). API routes are for *external* consumers (Module 20).

**BAD:** a 200-line `is:inline` script with jQuery-era DOM code.

**GOOD:** a bundled `<script>` module, or an island if it has component state.

**BAD:** `<div class="hero">` soup with no headings/landmarks.

**GOOD:** semantic elements (Module 28) — components are where semantics are enforced.

**BAD:** using `set:html` for CMS output without sanitizing.

**GOOD:** sanitize server-side (allow-list) before `set:html`, or use MDX/Markdown rendering.

## Security Notes

- Template expressions auto-escape; `set:html` never does (XSS).
- Frontmatter secrets stay server-side **unless** you import the file into an island — the bundler follows imports.
- `define:vars` serializes values **into client scripts** (JSON) — anything you pass is public.

## Performance Notes

- Components add build-time cost only; runtime cost is the HTML you render.
- Deep slot/prop chains are fine; don't invent wrapper components "for symmetry."
- Scoped CSS is per-component CSS files — trivial; don't fear `<style>`.

## Exercises

**Beginner.** Build `Stat.astro` taking `label` and `value` props; render three of them. Add scoped styles. Confirm zero JS in the built page.

**Intermediate.** Build `Tabs.astro` with progressive enhancement: all tab panels rendered as sections; a `<script>` upgrades to show/hide with correct `aria-selected` + arrow-key handling (a11y preview from Module 28). The page must still show all panels with JS disabled.

**Production.** Create `Notice.astro` with a `title` slot fallback and a `type` prop (`info|warn|error`) using `class:list` and CSS custom properties; document it with a JSDoc `@example`.

**Architecture Challenge.** You need a "copy code" button on every code block across 200 docs pages. Island per block? Compare with: *one global bundled `<script>` that delegates clicks on `[data-copy]` buttons — zero islands, works on all pages, ~1 KB.*

**Debugging Challenge.** Styles "leak" between two components — both use `.title` but you expected scoping. Found `<style is:global>` in a shared layout overriding them. Diagnose and fix (rename to scoped, or namespace global rules).

## MENTAL MODEL — Astro Component

> An `.astro` file is a **server-side render function with a template literal UI**. Frontmatter decides, template renders, style scopes, script enhances. It has no runtime, no state, and no rerenders — which is exactly why it is cheap. Choose an island only when the *interaction* demands component state.

## Official Documentation

- Astro syntax: https://docs.astro.build/en/reference/astro-syntax/
- Components: https://docs.astro.build/en/basics/astro-components/
- Scripts: https://docs.astro.build/en/guides/client-side-scripts/
- Styling: https://docs.astro.build/en/guides/styling/
- Template directives: https://docs.astro.build/en/reference/directives-reference/

## What I Should Know Before Continuing

1. When does frontmatter run relative to the browser? Can it `await`?
2. What is scoped in `<style>`, and how do you go global *deliberately*?
3. Difference between `<script>`, `<script is:inline>`, and `define:vars` regarding what ships to the client.
4. What changed in v7 regarding invalid HTML in templates?
