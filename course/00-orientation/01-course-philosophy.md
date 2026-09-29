# 01 — Course Philosophy & the Astro Mindset

> "Build as much as possible with HTML, CSS, Astro, and the web platform. Add JavaScript only where it provides real value."

## What This Course Is NOT

- **Not** "React, but with `.astro` files."
- **Not** a tour of syntax you'll forget in a week.
- **Not** a marketing brochure for Astro. Module 39 is literally titled *When Not to Use Astro*.
- **Not** a place where a state-management library is the answer to every problem.

## The Five Disciplines

### 1. Content-first

Before components, ask: **what is the content, and what are its URLs?** Astro is strongest when you model content as structured, validated, typed data (Content Layer) and let file-based routing mirror your information architecture. Design the schema and the routes first; the components follow.

### 2. HTML first, CSS second

The page must be a real document: semantic landmarks, real links, real forms, correct headings. CSS handles presentation, responsive layout, dark mode, and — increasingly — interactivity that once demanded JS (details/summary, :target, popover, dialog, :has, container queries, scroll-driven animations). **A `<details>` accordion is not a component problem.**

### 3. Astro third

Astro renders the document: components, layouts, dynamic routes, content queries, server data. `.astro` frontmatter runs **only on the server/build** and produces **HTML**. This is your default rendering tool.

### 4. JavaScript when necessary

If an interaction genuinely requires JS (live filtering, optimistic UI, drag-and-drop, canvas), write the **smallest script that works** — often a vanilla `<script>` (bundled, type-safe, tiny) rather than any framework. Astro bundles `<script>` tags by default; a 2 KB vanilla script beats a 40 KB island.

### 5. Client framework only when necessary

A UI framework island earns its bundle when the interaction has **real component state**: complex forms with derived state, data tables with many interacting controls, live-updating views, rich editors. Then React (this course's choice) is a great tool — **inside the island boundary**, never as the page architecture.

## The Escalation Ladder

Every feature climbs this ladder and stops at the first rung that suffices:

```
1. Static HTML semantics        (landmarks, links, tables, details)
2. CSS                          (layout, animation, :has, popover, transitions)
3. Platform APIs                (dialog, <form>, URL params, localStorage, fetch)
4. Vanilla <script> in Astro    (bundled TS, no framework)
5. Astro server capability      (Action, endpoint, server island, session)
6. Framework island             (client:load/idle/visible/media/only)
7. Multiple frameworks / global client state   (rare; justify explicitly)
```

Most features never leave rungs 1–3. That is the point.

## What "Architecture" Means Here

In this course, an architectural decision is any choice that answers: **where does this run, what does it cost, and who is responsible for its correctness?**

| Decision | Runs where | Cost paid | Correctness owner |
|---|---|---|---|
| Prerendered page | build time → CDN | build seconds | content pipeline |
| SSR page | request time | latency per request | server code + cache policy |
| Server island | request time, deferred | extra request | server code |
| Client island | browser | bytes + hydration + INP risk | island code + directive choice |
| Action | server, on demand | request | validation + authz + service |
| Middleware | server, every request | latency per request | security & context setup |

"Senior Astro engineer" means you can make this table for every feature in a spec — before writing code.

## Teach by Contrast (the course's house style)

**BAD:** entire React SPA mounted inside Astro.
**GOOD:** Astro page + React islands only where interaction lives.

**BAD:** `useEffect` + `fetch` to your own server to render a list of posts.
**GOOD:** `getCollection()` at build time — the list is already HTML.

**BAD:** global client store holding the current user's role for "authorization."
**GOOD:** role checked on the server at every Action; client store only mirrors UI state.

**BAD:** a 300 KB chart library hydrated on page load for a chart below the fold.
**GOOD:** `client:visible` on the chart island — or an `<img>` of a build-time rendered chart.

## How to Study This Course

1. Read a module once for the mental model.
2. Re-read and type the code samples (they are production patterns, not pseudo-code).
3. Do the **Beginner** and **Intermediate** exercises the same day.
4. Do the **Production** exercise within a week.
5. Attempt the **Architecture Challenge** *before* reading the provided review — the review is the actual lesson.
6. The **Debugging Challenge** teaches the failure modes you'll hit in real jobs. Do not skip it.

Each module ends with **"What I Should Know Before Continuing."** If you can't answer those questions, stop and repeat the module.

## MENTAL MODEL — Course Philosophy

> The browser's job is to render documents and run the interactions a user can actually feel. The server's job is to produce authoritative documents and enforce rules. Astro's job is to make the boundary explicit and let you pay for JavaScript one island at a time. If you can't name what a piece of code is buying the user, delete it.

## Official Documentation

- Astro docs home: https://docs.astro.build/en/
- Islands concept: https://docs.astro.build/en/concepts/islands/
- Why Astro: https://docs.astro.build/en/concepts/why-astro/

## What I Should Know Before Continuing

1. What are the five disciplines, and in what order do you apply them?
2. What is the escalation ladder? Which rung handles an accordion? A live search box? A comments section?
3. What makes something an "architectural decision" in this course?
4. Why is "React for everything" an architecture problem, not a syntax problem?
