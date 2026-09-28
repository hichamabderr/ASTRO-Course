# Module 04 — Props, Slots & Component Composition

> Phase 3 continued. How data flows into components — and how Astro's composition model differs from React's.

## Props: typed, server-side, plain

```astro
---
// src/components/ProductCard.astro
import { Image } from 'astro:assets';

interface Props {
  title: string;
  price: number;
  currency?: string;          // optional
  image: ImageMetadata;       // typed asset
  badge?: 'new' | 'sale';
}
const { title, price, currency = 'USD', image, badge } = Astro.props;
// Props are validated by TypeScript AT COMPILE TIME only — not at runtime.
---

<article class="card">
  {badge && <span class="badge">{badge}</span>}
  <Image src={image} alt="" width={400} height={300} />
  <h3>{title}</h3>
  <p>{price.toFixed(2)} {currency}</p>
  <slot name="actions" />
</article>
```

**Props facts:**
- Destructuring + defaults work exactly like plain functions (because it is one).
- Types are compile-time. **Runtime validation belongs at trust boundaries** (URL params, form input, APIs) with Zod — not on every component prop.
- Any JSON-serializable value works; functions/classes can be passed **between Astro components** (they run in the same process) but **cannot** cross into islands or server islands (serialization boundary!).

```astro
---
import ProductCard from './ProductCard.astro';
import hero from '../assets/hero.jpg';
---
<ProductCard title="Desk" price={249} image={hero} badge="new">
  <button slot="actions">Add to cart</button>
</ProductCard>
```

## Slots: composition the HTML way

```astro
---
// src/layouts/BaseLayout.astro
interface Props { title: string; description?: string }
const { title, description = '' } = Astro.props;
---

<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>{title}</title>
    <meta name="description" content={description} />
    <slot name="head" />          <!-- named slot: per-page extras -->
  </head>
  <body>
    <slot />                       <!-- default slot: page content -->
  </body>
</html>
```

**Named slots, fallbacks, and slot expressions:**

```astro
<!-- fallback content renders when the consumer passes nothing -->
<slot name="footer">
  <footer>© Default footer</footer>
</slot>

<!-- forward a slot through a wrapper -->
<section>
  <slot name="aside" />
</section>
```

**Scoped slots (render props, server-style):** slots can carry data back to children:

```astro
---
// ZebraList.astro
const items = ['a', 'b'];
---
<ul>
  {items.map((item, i) => (
    <li><slot name="item" item={item} index={i} /></li>
  ))}
</ul>

<!-- consumer -->
<ZebraList>
  <Fragment slot="item" let:item let:index>
    <strong>{index}:</strong> {item}
  </Fragment>
</ZebraList>
```

## Astro vs React composition — explicit comparison

| Concern | React | Astro |
|---|---|---|
| Children | `props.children` | `<slot />` |
| Named children | props convention | named slots |
| Data in | props (runtime, re-rendered) | props (per render, no re-render) |
| Render props | function props | scoped slots (`let:`) |
| State | `useState` etc. | **none** — URL/session/server/island state (Module 24) |
| Side effects | `useEffect` | fetch before render in frontmatter |
| Composition goal | reusable *behavior* | reusable *documents* |

The deep comparison lives in [Module 09](09-framework-interop-and-react-islands.md). The one-line version: **React composes behavior; Astro composes documents.**

## Layouts = components with slots

Layouts are ordinary components with a convention (wrapping the document). Pattern for per-page head control:

```astro
---
// src/layouts/BaseLayout.astro — see above
---
<!-- src/pages/about.astro -->
<BaseLayout title="About" description="Who we are">
  <Fragment slot="head">
    <link rel="canonical" href="https://example.com/about/" />
  </Fragment>
  <h1>About</h1>
</BaseLayout>
```

(Named-slot-head patterns become the SEO component in Module 26.)

## Common Mistakes

**BAD:** runtime-prop-validated every card with Zod `safeParse` in hot loops.

**GOOD:** Zod at boundaries (content schemas, `Astro.params`, form data); TS types inside.

**BAD:** passing a callback prop into a `client:load` React island and expecting it to work.

**GOOD:** islands receive **serializable props only** (strings, numbers, plain objects — no functions/class instances). Communicate via events/URL/servers (Module 07).

**BAD:** `<div class="slot-wrap">` wrappers everywhere "so styling is easier."

**GOOD:** style the projected content from the parent scope or use `:global` sparingly with clear naming (Module 06).

**BAD:** default slot soup — everything through one slot in a "mega layout."

**GOOD:** named slots (`head`, `aside`, `actions`) keep documents explicit.

## Security Notes

- Props from content schemas are validated (good). Props from `Astro.url`/requests are not — validate at the page level before rendering.
- Slot content is compiled in the *consumer's* scope; scoping IDs follow the authoring component. Don't rely on `:global` for user-supplied class names.

## Performance Notes

- Slots and props have **zero runtime cost** — composition is free in the browser.
- Avoid "render-prop islands": scoped slots keep composition on the server.

## Exercises

**Beginner.** Build `Badge.astro` (`tone`, `label?` props, default slot content). Render with and without slot content.

**Intermediate.** Build `BaseLayout` with `title`, `description`, `head` slot, default slot, and `footer` slot with fallback. Use it on three pages.

**Production.** Build `DataTable.astro` accepting `columns: {key,label}[]` and `rows: Record<string,string>[]`, with a `cell` scoped slot that receives `row` and `column` — consumers format cells their way.

**Architecture Challenge.** A design system page needs `<Button>` in React (for the Storybook-like playground island) and static docs pages. Should Button exist twice? Compare with: *YES — a 20-line Astro `Button.astro` for documents and the React button inside the playground island. "One implementation" is an anti-goal when it forces React onto static pages.*

**Debugging Challenge.** Scoped slot content doesn't show: consumer wrote `<p slot="item">` instead of `<Fragment slot="item" let:item>`. Explain why `let:` binding requires a fragment/element that can receive slot props, and fix.

## MENTAL MODEL — Composition

> Props are a server-render function's arguments; slots are its children — composition happens **at render time, for free**. React composition manages behavior lifecycles; Astro composition assembles documents. Cross the boundary (into islands) with serializable data only.

## Official Documentation

- Astro props/slots: https://docs.astro.build/en/basics/astro-components/
- Layouts: https://docs.astro.build/en/basics/layouts/
- TypeScript: https://docs.astro.build/en/guides/typescript/

## What I Should Know Before Continuing

1. Where is prop validation performed for component props? For trust boundaries?
2. What can't you pass to a client island, and why?
3. How do scoped slots differ from React render props in *where they execute*?
4. Why are layouts "just components"?
