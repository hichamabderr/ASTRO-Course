# Module 24 — Client-Side State & TanStack Query (Evaluation)

> Phase 20. **Do not blindly add global state.** State has an *owner*; the owner determines where the state lives.

## The state ownership ladder

```text
1. HTML state        open/closed, checked, selected — the DOM already has it
2. URL state         filters, search, pagination, tabs — shareable, back-button-safe
3. Server state      user, cart, permissions — authoritative; cache it
4. Island-local      dropdown open, form draft, animation frame
5. Shared island     two islands need the same changing value
6. Global client     many components, frequent updates, client-only app zone
```

| Kind | Lives in | Example | Tool |
|---|---|---|---|
| HTML | attributes/elements | accordion | `<details>`, CSS |
| URL | `?q=`, hash, path | search query | `Astro.url`, `history`, forms GET |
| Server | DB/session | cart contents | Actions, `context.session` |
| Island-local | component state | modal open | `useState` / `ref` |
| Shared island | module store | theme, filter sync | **vanilla module** or **nanostores** |
| Global client | app-zone store | dashboard data cache | zustand / TanStack Query |

## What React state is NOT for

- Anything the back button should restore → **URL**.
- Anything the server must trust (role, price, ownership) → **server** (client state is a cache).
- Anything static at build → **props/HTML**.

## Shared state across islands — the options

### 1. The DOM (free and underrated)

```ts
// theme: html[data-theme] is the store; islands read attributes via MutationObserver if needed
document.documentElement.dataset.theme = 'dark';
```

### 2. A shared vanilla module (no library)

```ts
// src/lib/store/searchStore.ts
type State = { q: string };
let state: State = { q: '' };
const subs = new Set<(s: State) => void>();
export const searchStore = {
  get: () => state,
  set(q: string) { state = { q }; subs.forEach((f) => f(state)); syncToUrl(q); },
  subscribe(f: (s: State) => void) { subs.add(f); return () => subs.delete(f); },
};
```

Both React islands import the same module → one shared truth, ~0.5 KB.

### 3. Nanostores (course's library pick *if* a library is justified)

Tiny, framework-agnostic (React/Vue/Svelte/Solid adapters) — the de-facto Astro islands store. Zustand is fine **inside React-only zones**.

**When justified:** ≥3 islands, frequent updates, no natural URL/DOM home. Before that: module store or URL.

## TanStack Query in Astro — evaluated honestly

TanStack Query is a **server-state cache for client-heavy UIs**: request dedupe, background refetch, mutations with optimistic updates, stale-while-revalidate, infinite queries, offline support.

**You usually DON'T need it in Astro** because:
- most data is rendered server-side (no client fetch at all),
- one-shot island data can be props or one action call,
- forms mutate via actions without cache invalidation puzzles.

**You SHOULD consider it inside an island (or app zone) when:**
- widgets **poll** (notifications, dashboards, live prices),
- complex **mutation + cache invalidation** graphs (like/follow across lists),
- **background refetch / focus revalidation** matters,
- offline/queueing is a requirement.

```tsx
// inside ONE React island (app zone) — the Astro site stays a site
import { QueryClient, QueryClientProvider, useQuery } from '@tanstack/react-query';
import { actions } from 'astro:actions';

const qc = new QueryClient();

function Notifications() {
  const { data } = useQuery({
    queryKey: ['notifications'],
    queryFn: () => actions.getNotifications().then((r) => r.data ?? []),
    refetchInterval: 30_000,
  });
  // ...
}

export function AppZone() {
  return <QueryClientProvider client={qc}><Notifications /></QueryClientProvider>;
}
```

`<AppZone client:load />` on `/app/*` pages only. **The public site never pays for it.**

## Common Mistakes

**BAD:** Redux/Zustand wrapping the whole Astro site via one root island.

**GOOD:** URL + server + island-local; stores only in app zones.

**BAD:** duplicating server truth (user role) into a store and authorizing with it.

**GOOD:** server authoritative; store mirrors display data with clear staleness.

**BAD:** `useEffect` fetch + `setState` + loading/error boilerplate in 12 components.

**GOOD:** TanStack Query *inside the app zone* — or one action + props if data is static per load.

**BAD:** syncing state between islands via `window.__APP__`.

**GOOD:** shared module/nanostores with typed API.

## Security Notes

- Client stores are readable/alterable (devtools). Never secrets, never authz.
- URL state can be poisoned (`?role=admin`): validate/ignore unknown params server-side.

## Performance Notes

- Stores are cheap; **subscriptions** cause renders — keep islands granular.
- TanStack Query = ~13 KB gzip-ish (measure current) — app-zone cost only.
- URL state = 0 KB and survives reload/share — always the first choice for filters.

## Exercises

**Beginner.** Filterable product list with state in URL (`?q=&sort=`); works on reload/share; 0 KB JS (GET form + server) or a tiny island.

**Intermediate.** Two islands (search box + results count) sharing a vanilla module store + URL sync.

**Production.** Notifications island polling with TanStack Query inside an app zone; document why the public pages don't include it.

**Architecture Challenge.** A SaaS with: marketing site, docs search, dashboard with 8 widgets, and a global command palette. Lay out the state plan per zone.

> **Review:** marketing: none; docs search: URL + one island; dashboard widgets: TanStack Query in app zone (or per-widget actions + a light store); command palette: island-local + URL for executed commands. One global store is unnecessary.

**Debugging Challenge.** Filter island resets on back navigation. State lived in `useState` only. Fix: read initial state from `URLSearchParams`, write updates with `history.replaceState` + listen `popstate` (and `astro:page-load` if using ClientRouter).

## MENTAL MODEL — State Ownership

> Every piece of state has exactly one owner: **DOM, URL, server, island, store — in that order of preference.** Choose the owner first; the library last. If you can't say who owns it, the answer is the server or the URL.

## Official Documentation

- Sharing state recipe: https://docs.astro.build/en/recipes/sharing-state/
- Islands: https://docs.astro.build/en/concepts/islands/
- TanStack Query: https://tanstack.com/query/latest
- Nanostores: https://nanostores.dev/

## What I Should Know Before Continuing

1. The six-rung ladder. Where does a wizard's step counter live? A live chat? A theme?
2. Why is URL state the performance default for filters?
3. When does TanStack Query earn its bytes in an Astro app?
4. How do two React islands share state without a library?
