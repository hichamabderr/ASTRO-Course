# Reference — Request Flows & Sequence Diagrams (§81–82)

> Mermaid diagrams (GitHub renders these). Print the four flows mentally until they're instinct.

## 1. The four request flows

### Static page request

```text
Browser → CDN → (HIT) HTML bytes → paint → (islands hydrate per directive)
                → (MISS) origin file → cache → HTML bytes
```

```mermaid
sequenceDiagram
  participant B as Browser
  participant CDN as CDN edge
  participant O as Static origin (dist/)
  B->>CDN: GET /blog/astro-islands/
  alt cache hit
    CDN-->>B: 200 HTML (cached)
  else miss
    CDN->>O: fetch file
    O-->>CDN: index.html
    CDN-->>B: 200 HTML
  end
  B->>B: paint (LCP) — JS only for islands
```

### SSR request

```mermaid
sequenceDiagram
  participant B as Browser
  participant RT as Adapter runtime
  participant MW as Middleware
  participant P as Page frontmatter
  participant DB as Service/DB
  B->>RT: GET /app/dashboard (cookie)
  RT->>MW: onRequest
  MW->>MW: requestId, security headers, session → locals.user
  MW->>P: next()
  P->>DB: scoped queries
  DB-->>P: rows
  P-->>MW: HTML
  MW->>MW: Cache-Control: private, no-store
  MW-->>B: 200 HTML
```

### Island hydration

```mermaid
sequenceDiagram
  participant B as Browser
  participant D as Document HTML
  participant I as Island chunk + framework
  B->>D: parse HTML (server-rendered island DOM + loader script)
  D-->>B: paint
  Note over B: directive schedules (load/idle/visible/media)
  B->>I: dynamic import()
  I-->>B: chunk + shared runtime
  B->>B: hydrate() — attach listeners/state to existing DOM
```

### Action invocation (form → response → UI)

```mermaid
sequenceDiagram
  participant F as Form / island
  participant MW as Middleware
  participant A as Action handler
  participant S as Service
  participant DB as Database
  F->>MW: POST (action path, cookie)
  MW->>MW: locals.user (or null)
  MW->>A: route to action
  A->>A: Zod input validation
  alt invalid
    A-->>F: BAD_REQUEST + field errors (typed)
  else valid
    A->>A: authN (401 if anonymous)
    A->>S: command + actor
    S->>S: authZ (role + ownership + tenant)
    S->>DB: transactional write
    DB-->>S: rows
    S-->>A: DTO
    A-->>F: data (or ActionError)
  end
```

## 2. Extended sequence diagrams

### Authentication (login)

```mermaid
sequenceDiagram
  participant B as Browser
  participant E as /api/auth handler
  participant DB as Users/Sessions
  B->>E: POST email+password
  E->>DB: verify credentials (constant-time)
  alt valid
    E->>DB: create session (rotate id)
    E-->>B: Set-Cookie: session (HttpOnly; Secure; SameSite) + redirect /app
  else invalid
    E-->>B: 401 generic message
  end
```

### Logout

```mermaid
sequenceDiagram
  participant B as Browser
  participant MW as Middleware
  participant A as Action logout
  participant DB as Sessions
  B->>MW: POST logout (cookie)
  MW->>A: locals.user set
  A->>DB: destroy session / revoke
  A-->>B: Set-Cookie cleared + redirect /
```

### Protected Action (createOrder)

```mermaid
sequenceDiagram
  participant I as Cart island
  participant A as actions.createOrder
  participant S as OrderService
  participant DB as Postgres
  I->>A: call (idempotency key)
  A->>A: validate input, authN
  A->>S: createOrder(actor, cart, key)
  S->>DB: tx: insert order, decrement stock (FOR UPDATE)
  alt stock conflict
    DB-->>S: constraint error
    S-->>I: ActionError CONFLICT
  else ok
    DB-->>S: order
    S-->>I: DTO + redirect target
  end
```

### Server endpoint (public API)

```mermaid
sequenceDiagram
  participant C as External client
  participant E as GET /api/v1/posts
  participant DB as DB
  C->>E: GET ?limit=20&cursor=…
  E->>E: Zod-validate query, rate-limit
  E->>DB: scoped, indexed query
  E-->>C: 200 JSON (Cache-Control public, max-age=60)
```

### File upload

```mermaid
sequenceDiagram
  participant U as User
  participant A as uploadAvatar action
  participant V as Validator (magic bytes/pixels)
  participant S3 as Object storage
  participant DB as DB
  U->>A: multipart file
  A->>A: size cap (z.instanceof(File).refine)
  A->>V: sniff + probe + re-encode
  A->>S3: put(uuid.webp)
  A->>DB: store key + owner
  A-->>U: { url }
```

### Webhook (payments)

```mermaid
sequenceDiagram
  participant P as Payment provider
  participant W as POST /api/webhooks/stripe
  participant DB as WebhookEvents
  participant Q as Queue/worker
  P->>W: payload + signature
  W->>W: verify signature over RAW body (timestamp tolerance)
  W->>DB: alreadyProcessed(event.id)?
  alt duplicate
    W-->>P: 200 (no-op)
  else new
    W-->>P: 200 fast
    W->>Q: process async (fulfill order)
  end
```

### Middleware pipeline (ordering)

```mermaid
flowchart LR
  R[Request] --> L[logging / requestId]
  L --> SEC[security headers + origin check]
  SEC --> SES[session → locals.user/tenant]
  SES --> RT{route class}
  RT -->|page/action/endpoint| NEXT[next → handler]
  RT -->|webhook| WH[signature auth path]
  NEXT --> OUT[response: cache headers, logs]
  WH --> OUT
```

### Tenant authorization (the isolation proof)

```mermaid
sequenceDiagram
  participant U as User A (org-1)
  participant A as Action getProject
  participant S as ProjectService
  participant DB as Postgres
  U->>A: getProject(id = org-2's project)
  A->>S: getForActor(id, actor)
  S->>DB: WHERE id=$1 AND org_id=$2 (actor.org)
  DB-->>S: 0 rows
  S-->>U: 404 not_found (no existence leak)
```

### View transition (client navigation)

```mermaid
sequenceDiagram
  participant B as Browser (ClientRouter)
  participant N as Next page
  B->>B: click intercepted (same-origin link)
  B->>N: fetch HTML (astro:before-preparation)
  N-->>B: document
  B->>B: swap DOM (astro:before-swap / after-swap) — persist kept
  B->>B: document.startViewTransition (fade/morph)
  B->>B: astro:page-load → re-init scripts
```

## Usage drill
Blank-page drill: redraw the Action and Tenant diagrams from memory. If you can't, re-read Modules 16 and 22.
