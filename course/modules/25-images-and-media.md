# Module 25 — Images & Media

> Phase 38. The current `astro:assets` system: optimization, responsive layouts, remote images, and when a plain `<img>` is enough.

## Concept: images are a build-time optimization problem

LCP is usually an image. Astro's asset pipeline (sharp-based) resizes, re-formats (webp/avif), fingerprints, and emits `srcset`/`sizes` — **at build** — so the browser downloads the right bytes.

## The API surface (current)

```astro
---
import { Image, getImage } from 'astro:assets';
import hero from '../assets/hero.jpg';        // local: optimized automatically
---

<Image src={hero} alt="Team at work" width={1200} height={630} />
<Image src={hero} alt="" layout="responsive" />   <!-- responsive behavior (see below) -->
```

| Feature | How |
|---|---|
| Local images | import from `src/assets/` → `<Image>` |
| Remote images | pass URL + dimensions, or `inferSize()`; allow-list via `image.domains`/`image.remotePatterns` |
| Multiple formats/sizes | `<Picture>` (art direction, formats fallback) |
| In content schemas | `image()` helper from `astro:content` (Module 12) |
| Raw URL at build | `getImage()` → `srcset` string for custom markup |
| SVG as component | `import Logo from './logo.svg'` → `<Logo />` (since 5.7) |
| Responsive | `layout` prop (`responsive`/`fixed`/`full-width`/`none`) + `image.layout`/`image.responsiveStyles` global config — auto `srcset`+`sizes`; evolved from the 5.10 experimental flags to current defaults (check the [images guide](https://docs.astro.build/en/guides/images/) for your version's exact config keys) |

```astro
---
import { Picture } from 'astro:assets';
import hero from '../assets/hero.jpg';
---
<Picture
  src={hero}
  widths={[400, 800, 1200]}
  sizes="(max-width: 800px) 100vw, 1200px"
  formats={['avif', 'webp']}
  alt="Conference keynote"
/>
```

## Decision: Astro image vs plain `<img>` vs `public/`

| Situation | Choice |
|---|---|
| Local raster, known at build | `<Image>` / `<Picture>` |
| CMS image URL | `<Image>` with `inferSize` or fixed dims + remotePatterns allow-list |
| Tiny icon | SVG component or inline SVG |
| Animated GIF as brand moment | `public/` as-is (optimization would kill animation) or `<Image>` if you accept conversion |
| OG image | build-time generated (static file or `getImage` at build) |
| User upload (runtime) | object storage + resizing service — **not** `astro:assets` (build-time!) |

## Rules of the road

1. **Always set dimensions** (or `inferSize`) — CLS depends on it.
2. **`alt` is content:** describe function; `alt=""` for decorative.
3. **Lazy-load below-fold images** (`loading="lazy"`; `<Image>` handles patterns; be careful with LCP hero — *don't* lazy it; consider `fetchpriority="high"`).
4. **Preload only the LCP image** (one, not five).
5. Prefer **avif/webp** via `<Picture>` for large photography; keep legacy fallback if analytics show ancient browsers.

## Media beyond images

- **Video:** never autoplay with sound; use facade thumbnails + click-to-load iframes (privacy + perf).
- **Fonts (adjacent perf trap):** self-host subset woff2, `font-display: swap/optional`; Astro Fonts API (stable since v6) handles preload/fallback optimization if you adopt it.
- **Audio/embeds:** lazy after consent where required (Module 31).

## Common Mistakes

**BAD:** 4K camera JPEG in `public/` as a hero.

**GOOD:** `src/assets` + `<Image width={1600}>` → 200 KB avif.

**BAD:** `loading="lazy"` on the LCP hero.

**GOOD:** eager + `fetchpriority="high"` for the one hero.

**BAD:** remote images from any user-supplied URL ("flexibility").

**GOOD:** `remotePatterns` allow-list (SSRF/phishing surface — Module 23).

**BAD:** building a client-side lightbox island for every gallery.

**GOOD:** `<dialog>` + CSS or one shared script; islands only for complex media UIs.

## Security Notes

- Validate upload pipelines (Module 36): re-encode, strip EXIF, cap pixels (decompression bombs).
- Remote image config is a security control — treat as allow-list, review as code.
- `alt`/captions from CMS: escaped by default; don't `set:html` captions.

## Performance Notes

- Images dominate page weight; `srcset`+right formats routinely cut 60–80%.
- Measure LCP element; ensure it's the optimized hero and preloaded.
- Set explicit cache headers for `/_astro/` hashed assets (immutable).

## Exercises

**Beginner.** Blog post hero via `<Image>` with correct dims + alt; compare bytes to the raw file.

**Intermediate.** `<Picture>` gallery with `widths` + `sizes`; verify emitted `srcset` in `dist` HTML; test CLS = 0 with dimensions.

**Production.** Remote CMS images through `remotePatterns`; an OG image generated at build; LCP tuned (preload + no lazy) on the homepage — measure with Lighthouse before/after.

**Architecture Challenge.** Marketplace where sellers upload photos at runtime. Where does `astro:assets` fit — and where does it *not*?

> **Review:** `astro:assets` is build-time → seller uploads go to object storage + runtime image service (signed URLs, resize params); the *listing templates* still use `<Image>` for static chrome; upload validation per Module 36.

**Debugging Challenge.** `<Image src={remoteUrl} />` throws in build: missing dimensions and untrusted remote. Fix: `inferSize()` or explicit `width/height` + add host to `image.domains`/`remotePatterns`; explain why the allow-list exists.

## MENTAL MODEL — Images

> Treat images as **build artifacts** wherever possible: declared in code or content schemas, optimized once, served immutable, sized to their slot. Runtime media (uploads) is a storage+validation problem, not a bundler problem. The hero image is the LCP timer — design for it first.

## Official Documentation

- Images guide: https://docs.astro.build/en/guides/images/
- Image reference: https://docs.astro.build/en/reference/modules/astro-assets/
- Fonts: https://docs.astro.build/en/guides/fonts/

## What I Should Know Before Continuing

1. `<Image>` vs `<Picture>` vs `getImage` vs `public/` — selection table.
2. Three CLS/LCP rules for images.
3. Why can't runtime user uploads flow through `astro:assets`?
4. What do `remotePatterns` protect against?
