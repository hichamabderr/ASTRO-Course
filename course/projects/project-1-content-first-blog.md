# Project 1 — Content-First Blog ("Modern Docs + Blog Platform", part 1)

> Modules applied: 03–06, 10–13, 25–26, 28–29. Deliverable: a production-shaped blog you would put in a portfolio.

## Feature list

- Homepage (featured + recent posts)
- Blog index with **pagination** (`/blog/`, `/blog/page/2/`)
- Post pages: hero image, author, tags, TOC from headings, related articles
- Category + tag archive pages
- Author pages
- MDX "tutorial" posts with callouts + one interactive demo island
- Search (static index — Module 35 pattern)
- RSS + sitemap + `robots.txt`
- Full SEO layer (Module 26 `SEO.astro` + JSON-LD Article + OG images)
- Dark mode (token swap — Module 06)
- Responsive, accessible (skip link, landmarks, contrast — Module 28)
- **Zero framework JS** except: (1) search island (`client:idle`), (2) demo island (`client:visible`)

## Architecture

```text
src/
  content/blog/**.md(x)        content.config.ts (posts, authors, categories)
  pages/index.astro            pages/blog/[...slug].astro   pages/blog/page/[page].astro
  pages/tags/[tag].astro       pages/authors/[id].astro     pages/categories/[id].astro
  pages/search.astro           pages/rss.xml.ts
  components/SEO.astro  PostCard.astro  TOC.astro  Pagination.astro
  components/react/Search.tsx  (island)
  layouts/BaseLayout.astro
```

## Milestones

1. **M1 — Skeleton:** layout, tokens, dark mode, home + 3 posts from a collection. *(Modules 03–06)*
2. **M2 — Content model:** authors/categories references, draft filtering, TOC, related posts. *(10–12)*
3. **M3 — Routes & pagination:** archives, tag pages, `getStaticPaths` discipline. *(05, 13)*
4. **M4 — SEO & feeds:** SEO component, JSON-LD, sitemap, RSS, canonicals. *(25–26)*
5. **M5 — Islands:** search (URL state), one MDX demo. *(07–08, 11)*
6. **M6 — Quality:** a11y pass, Lighthouse ≥ 95, `astro check` + 5 Vitest tests. *(28–30)*

## Definition of done

- [ ] `astro build` output contains zero framework JS except the two islands' chunks
- [ ] LCP < 2.5s (mobile throttle) on home + post pages; CLS = 0
- [ ] RSS validates; sitemap excludes drafts; JSON-LD passes rich-results test
- [ ] Keyboard-only navigation works; axe: no serious/critical
- [ ] Search works without JS (GET form) and is instant with JS

## Stretch

- Reading-time + progress bar (tiny script, not an island)
- `updatedDate` → changelog page + `lastmod`
- i18n second locale for the 5 key pages (Module 35)

## Grading rubric (self-review)

| Criterion | Weight |
|---|---|
| Content model quality (schemas, references) | 20 |
| Rendering discipline (all static, justified islands) | 20 |
| SEO completeness | 20 |
| Performance (measured) | 20 |
| A11y + progressive enhancement | 20 |
