# Module 01 — The Astro Mental Model

> Phase 1. The single most important module. Everything else is detail layered on top of this.

## Concept: What IS Astro, mechanically?

Astro is a **server-first web framework that compiles components into static HTML**, and lets you inject interactive "islands" where — and only where — interaction is required.

**Where does code run?**

| Code | Runs | Output |
|---|---|---|
| `.astro` frontmatter (between `---`) | **Build time or server request time** | disappears — variables for the template |
| `.astro` template | compiles to | **HTML string** (plus CSS) |
| `<script>` in `.astro` | **browser** (bundled by Vite unless `is:inline`) | small JS file |
| `client:*` island | **browser** (hydrates) | framework runtime + component JS |
| `src/pages/**.ts` endpoint / Actions / middleware | **server** | `Response` |

There is no Astro runtime shipped to the browser. "Astro" as a framework exists at build/request time. The browser sees documents.

## Mental Model 1 — Two rendering pipelines

**Traditional SPA:**

```text
Browser
  ↓ downloads framework + app bundle (hundreds of KB)
JavaScript
  ↓ boots router, fetches data
Framework
  ↓ constructs virtual DOM
Render UI          ← first paint waits for ALL of this
```

**Astro:**

```text
Request
  ↓
Astro (build time OR server)
  ↓ runs frontmatter: content queries, data, logic
HTML + CSS          ← browser paints THIS immediately
  ↓
Browser
  ↓ executes only the island scripts you opted into
Interactive islands hydrate where needed
```

The document is the product. JavaScript is a **progressive enhancement budget**, spent island by island.

## Mental Model 2 — The page is a tree of disappearing components

```text
            ASTRO PAGE
                │
     ┌──────────┼──────────┐
     │          │          │
  Content     HTML      Layout
     │          │          │
     └──────────┼──────────┘
                │
        Static / SSR HTML
                │
             Browser
                │
      ┌─────────┼─────────┐
      │         │         │
    No JS   Island A   Island B
                │         │
             React     React/Vue/
                       Svelte/Solid
```

Astro components compose into the HTML tree and **do not exist at runtime**. Islands are the only persistent client components.

## Architecture — the request lifecycle

### A. Prerendered page (build time)

```text
astro build ──► page frontmatter runs ONCE ──► HTML file written to dist/
deploy ──► CDN serves file ──► browser paints
```

### B. On-demand page (SSR)

```text
browser request ──► adapter runtime ──► middleware ──► page frontmatter runs
              ──► HTML response ──► browser paints
```

### C. Island hydration (either mode)

```text
HTML arrives with island's server-rendered HTML + tiny directive script
  ↓ browser decides WHEN per directive (load/idle/visible/media)
  ↓ downloads island's JS chunk + framework chunk (shared)
  ↓ hydrate(): attach event listeners to existing DOM
  ↓ island is interactive
```

## Code: your first mental-model demo

```astro
---
// src/pages/index.astro
// This entire block runs on the server (or at build). It VANISHES.
import Layout from '../layouts/Layout.astro';
import SearchBox from '../components/react/SearchBox.tsx';

const title = 'Documents first';
const posts = [
  { slug: 'a', title: 'HTML is a feature' },
  { slug: 'b', title: 'Pay per interaction' },
];
// ↑ imagine these came from getCollection('blog') — Module 10
---

<Layout {title}>
  <h1>{title}</h1>

  <!-- Static markup: zero JS for eternity -->
  <ul>
    {posts.map((post) => (
      <li><a href={`/blog/${post.slug}/`}>{post.title}</a></li>
    ))}
  </ul>

  <!-- One interactive widget: JS budget spent HERE, on purpose -->
  <SearchBox client:load />
</Layout>
```

What the browser receives: full list HTML (crawlable, instant) + one hydrated island. What it does **not** receive: any code from the frontmatter, Astro's internals, or JS for the list.

## Common Mistakes

**BAD:** treating `.astro` like a React component ("I'll add state and rerender").

```astro
---
let count = 0; // ❌ this is not component state; it's build-time dead code
---
<button onclick="count++">{count}</button> <!-- ❌ never updates; broken model -->
```

**GOOD:** static output + an island (or plain script) for the interactive bit.

```astro
<button id="count">0</button>
<script>
  let count = 0;
  const btn = document.getElementById('count')!;
  btn.addEventListener('click', () => (btn.textContent = String(++count)));
</script>
```

**BAD:** a client `fetch()` in `useEffect` to load data the server already had.

**GOOD:** fetch in frontmatter (build/server), render HTML, ship it.

**BAD:** assuming frontmatter runs in the browser.

**GOOD:** frontmatter = server-only. If you paste a secret there "for the example," it never reaches the browser *via HTML* — but it can leak via careless imports into islands (Security Notes).

## Security Notes

- Frontmatter is server-side: secrets *belong* there — but anything **imported into an island** is bundled for the browser. The boundary is the **import graph of island files**, not the file extension.
- `{expression}` in templates is HTML-escaped automatically. `set:html` is **not** — XSS risk (Module 23).
- Never trust `Astro.url` search params or headers without validation in SSR.

## Performance Notes

- First paint does not wait for hydration. That's the LCP advantage over SPA boot.
- Every island costs: framework chunk (shared across islands) + component chunk + hydration CPU. Two islands of React share one React runtime download.
- Static HTML can be CDN-cached globally; keep pages cacheable whenever personalization allows (see Module 34).

## Exercises

**Beginner.** Build `src/pages/lesson.astro` that renders a list of three books from an array in frontmatter. Confirm with "View Source" that the list is plain HTML. Add a `<button>` counter with a vanilla `<script>`.

**Intermediate.** Add a React `<LikeButton client:idle />` island to the same page. Measure (Network panel): which JS files load, and when do they start?

**Production.** Make the book list come from `getCollection()` of a `books` collection (peek at Module 10 if needed; otherwise use a JSON import). Verify `astro build` output contains fully rendered HTML in `dist/`.

**Architecture Challenge.** A teammate proposes: "Let's make a `<BookList>` React component and hydrate the whole page so filtering works later." Write a 5-line response proposing the island boundary and where filter state should live (URL params + server or island-local). Compare with the review below.

> **Review:** Filtering is interaction → island (or even vanilla script + `<form>` GET). The list itself is content → Astro. Filter state that should survive share/refresh → **URL query params**. Ship `BookList` as an Astro component with an optional filter island above it; hydrate only the filter controls.

**Debugging Challenge.** The counter button appears but clicks do nothing. You wrote `<button onclick="handle()">` with `handle` defined inside frontmatter. Diagnose (frontmatter isn't in the browser) and fix with a real `<script>`.

## MENTAL MODEL — Astro Components

> An `.astro` file is a **template function that runs on the server and returns HTML**. Frontmatter is its body; the template is its return value. Components evaporate at runtime. Islands are foreign objects with a lifecycle — expensive, deliberate, and local. When in doubt: does this need to *respond to the user*, or just *be read by the user*?

## Official Documentation

- Why Astro: https://docs.astro.build/en/concepts/why-astro/
- Islands: https://docs.astro.build/en/concepts/islands/
- Astro components: https://docs.astro.build/en/basics/astro-components/
- Pages: https://docs.astro.build/en/basics/pages/
- Scripting: https://docs.astro.build/en/guides/client-side-scripts/

## What I Should Know Before Continuing

1. Where does frontmatter run? What does it output?
2. Draw both pipelines (SPA vs Astro) from memory. Where is first paint in each?
3. What exactly ships to the browser for (a) an Astro component, (b) a `client:load` island?
4. Why is "React state" the wrong model for an `.astro` file?
