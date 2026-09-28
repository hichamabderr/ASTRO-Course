# Module 11 — Markdown & MDX

> Phase 10. How content becomes HTML: the **Sätteri** pipeline (v7 default), the unified/remark/rehype escape hatch, MDX for interactive docs, and syntax highlighting.

## Concept: the Markdown build pipeline

```text
.md file ──► frontmatter parse ──► Markdown pipeline ──► HTML string + metadata
                                                     └─► rendered via render(entry)
```

**Astro 7 default: Sätteri** — Astro's native Rust Markdown pipeline. GFM + SmartyPants behavior preserved; faster builds. `@astrojs/markdown-remark` (the unified/remark/rehype stack) is **no longer installed by default**.

**OLD vs MODERN pipelines:**

| Era | Pipeline | Plugins |
|---|---|---|
| Astro ≤ 6 | unified (remark/rehype) always | `markdown.remarkPlugins`, `markdown.rehypePlugins` |
| Astro 7 | **Sätteri by default** | Sätteri MDAST/HAST plugins; unified only if you opt in |

```js
// astro.config.mjs — opt back into unified plugins if you depend on them
import { defineConfig } from 'astro/config';
import { unified } from '@astrojs/markdown-remark';

export default defineConfig({
  markdown: {
    processor: unified(),
    // remarkPlugins / rehypePlugins / remarkRehype work — but are deprecated paths
  },
});
```

Rule: **prefer Sätteri plugins** for new work; install `@astrojs/markdown-remark` only for legacy plugin needs.

## Frontmatter → typed data

Markdown files in a collection get schema validation (Module 10). Standalone `.md` pages also work (`src/pages/post.md`) and can use `layout` frontmatter:

```markdown
---
layout: ../../layouts/BaseLayout.astro
title: 'Hello'
---
Body **markdown** here.
```

Collection-based content (recommended) renders via `render(entry)` — giving you `Content`, `headings`, `remarkPluginFrontmatter`.

## Syntax highlighting

Built-in **Shiki** at build time (zero client JS):

```js
export default defineConfig({
  markdown: {
    shikiConfig: {
      theme: 'github-dark',
      wrap: true,
      // themes: { light: 'github-light', dark: 'github-dark' } for dual themes
    },
  },
});
```

Code blocks ship as styled HTML + CSS. No highlight.js in the browser. For fancy features (line numbers, titles, diff highlighting), use MDX components or Shiki transformers — still build-time.

## MDX: Markdown + components

```bash
npx astro add mdx
```

```mdx
---
title: 'Interactive docs'
---
import Callout from '../../components/Callout.astro';
import DemoCounter from '../../components/react/DemoCounter';

# Feature X

<Callout type="warn">Careful with `client:*` here.</Callout>

<DemoCounter client:visible />

```js
// highlighted code still works
export const x = 1;
```
```

**Markdown vs MDX:**

| | Markdown | MDX |
|---|---|---|
| Authoring | pure content | content + JSX components |
| Components | none (custom tags via `components` mapping in `render`) | import directly |
| Compile | Sätteri/unified only | MDX compiler + Astro |
| Use | blogs, docs, changelogs | docs with demos, interactive tutorials, design system docs |

**Directives note:** `client:*` is not valid inside MDX on arbitrary components. Wrap: create `DemoCounterIsland.astro` that imports the React component with `client:visible`, then use the wrapper in MDX.

## Custom component mapping in Markdown

```astro
---
import { render } from 'astro:content';
const { Content } = await render(entry);
import CustomH2 from '../components/CustomH2.astro';
---
<Content components={{ h2: CustomH2 }} />
```

(Recall v6+ note: passing `client:*` through `components` is unsupported — wrap in `.astro`.)

## The build pipeline, end to end

```text
content.config.ts declares blog (glob loader)
  → astro sync/dev: files parsed → schema-validated → stored (id per file)
  → page: getCollection('blog') → getStaticPaths → routes
  → render(entry): Sätteri → HTML → <Content /> in your layout
  → output: static HTML with highlighted code; zero client JS
```

## Common Mistakes

**BAD:** copying `remarkPlugins: [emoji]` from an old blog into v7 without `@astrojs/markdown-remark`.

**GOOD:** check the [Markdown guide](https://docs.astro.build/en/guides/markdown-content/) for the current default; port plugins or opt into `unified()` deliberately.

**BAD:** client-side Markdown parsing (marked/marked-it in the browser) for blog content.

**GOOD:** content is compiled at build. The browser receives HTML.

**BAD:** MDX for 500 plain articles (compile cost + authoring friction).

**GOOD:** `.md` for prose; `.mdx` only where components are needed.

**BAD:** `set:html` with a third-party markdown lib output.

**GOOD:** the pipeline (or MDX) with sanitization handled by it.

## Security Notes

- Markdown pipelines execute plugins at **build time** — a malicious plugin = build-time RCE; pin & audit plugin deps.
- MDX is JSX: **never** render untrusted user MDX (it's code execution at build). User content → sanitize + `set:html` (allow-list) or a restricted renderer.
- Watch `href` in content: enforce `rel="noopener"` for external links in your components; disallow `javascript:` URLs via rehype/Sätteri sanitization if authors aren't fully trusted.

## Performance Notes

- Build-time highlighting costs build seconds, ships 0 JS — the right trade.
- MDX pages compile slower than `.md`; keep heavy demos as components loaded lazily (`client:visible`).
- Inline critical CSS for code themes or share one theme CSS file globally.

## Exercises

**Beginner.** Two `.md` files in a collection, custom Shiki theme, rendered with headings list.

**Intermediate.** MDX docs page using `Callout.astro` + a `client:visible` React demo wrapped in an `.astro` island file.

**Production.** Add a Sätteri (or unified, documented why) plugin that auto-adds `rel="noopener noreferrer"` to external links; cover it with a unit test (Module 30).

**Architecture Challenge.** Docs with 1,200 pages, 30 of which need interactive demos; authors are non-engineers. `.mdx` everywhere? Compare with: *`.md` default; demos isolated in small `.mdx` sections or embedded via custom shortcodes; authoring friction and build time stay low.*

**Debugging Challenge.** `remark-gfm` table styles broken after v7 upgrade — you're on Sätteri which includes GFM, but your custom CSS targeted `.remark-table` classes from an old rehype plugin. Diagnose pipeline difference; either port the plugin or fix CSS for Sätteri output.

## MENTAL MODEL — Markdown

> Markdown is a **build-time compiler target**, not a runtime format. Frontmatter is data, body is template, plugins are build code, highlighting is precomputed. MDX is "Markdown that can import islands" — powerful, therefore rationed.

## Official Documentation

- Markdown content: https://docs.astro.build/en/guides/markdown-content/
- MDX: https://docs.astro.build/en/guides/integrations-guide/mdx/
- Syntax highlighting: https://docs.astro.build/en/guides/syntax-highlighting/
- v7 Markdown changes: https://docs.astro.build/en/guides/upgrade-to/v7/#new-default-markdown-processor-s%C3%A4tteri

## What I Should Know Before Continuing

1. What is the default Markdown pipeline in Astro 7? How do you opt into remark/rehype?
2. Markdown vs MDX: selection criteria?
3. Why can't `client:live` be applied inside MDX directly?
4. Where does syntax highlighting run, and what ships to the browser?
