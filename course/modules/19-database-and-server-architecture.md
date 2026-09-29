# Module 19 — Database & Server Architecture

> Phases 18, 29. PostgreSQL + Drizzle (course standard, post-`@astrojs/db` world), and the layered architecture: **Action/Endpoint → Service → Repository → DB**.

## Concept: the stack after `@astrojs/db` ⛔

Astro 7 removed `@astrojs/db`. Official guidance: use a database library directly — **Drizzle ORM**, `node:sqlite`, Turso (libSQL), Neon, PlanetScale, or plain SQL clients. This course standardizes on:

```text
PostgreSQL (Neon/Supabase/self-hosted) + Drizzle ORM + connection pooling
```

Why Drizzle: SQL-shaped, typed schema in TS, lightweight (no codegen daemon), works on Node + edge runtimes, and was the schema model `@astrojs/db` itself was built on.

## Schema & migrations

```ts
// src/server/db/schema.ts
import { pgTable, text, timestamp, uuid, pgEnum, index } from 'drizzle-orm/pg-core';

export const roleEnum = pgEnum('role', ['user', 'admin', 'owner']);

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  passwordHash: text('password_hash').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const posts = pgTable('posts', {
  id: uuid('id').defaultRandom().primaryKey(),
  authorId: uuid('author_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  title: text('title').notNull(),
  body: text('body').notNull(),
  publishedAt: timestamp('published_at'),
}, (t) => [index('posts_author_idx').on(t.authorId)]);
```

```bash
npx drizzle-kit generate      # SQL migrations from schema diff
npx drizzle-kit migrate       # apply
```

Migrations are **files in git**; apply them in CI/CD before deploy. Never mutate prod schema by hand.

## Connection management (the #1 serverless foot-gun)

```ts
// src/server/db/client.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';

// Serverless: limit pool size; one client per instance
const queryClient = postgres(process.env.DATABASE_URL!, { max: 10 });
export const db = drizzle(queryClient);
```

Rules:
- **Never** create a pool per request.
- On edge/serverless, prefer driver built for it (Neon serverless driver, `postgres-js`, `@libsql/client`) and `max` small (1–10).
- Long-lived Node (`standalone`): module-level pool is fine (10–20).

## CRUD + transactions + pagination

```ts
// src/server/repositories/postRepository.ts
import { db } from '../db/client';
import { posts } from '../db/schema';
import { and, desc, eq, lt } from 'drizzle-orm';

export const postRepository = {
  listByAuthor(authorId: string, cursor?: string, limit = 20) {
    return db.select().from(posts)
      .where(and(eq(posts.authorId, authorId), cursor ? lt(posts.id, cursor) : undefined))
      .orderBy(desc(posts.id)).limit(limit + 1);   // +1 to detect next page
  },
  create(values: typeof posts.$inferInsert) {
    return db.insert(posts).values(values).returning();
  },
  async transfer(fromId: string, toId: string, cents: number) {
    return db.transaction(async (tx) => {
      await tx.execute(/* UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1 */);
      await tx.execute(/* credit */);
    });
  },
};
```

## Layering: Action → Service → Repository

```text
src/actions/index.ts        thin: validate + authz + call service
src/server/services/        business rules, transactions, orchestration
src/server/repositories/    SQL/ORM only
src/server/db/              client + schema + migrations
```

```ts
// src/server/services/commentService.ts
export async function createComment(input: { postId: string; body: string }, user: User) {
  const post = await postRepository.get(input.postId);
  if (!post || (post.draft && user.role === 'user')) throw new DomainError('not_found');
  return commentRepository.create({ ...input, authorId: user.id });
}
```

**Why not put SQL in pages?** Pages change for presentation reasons; queries change for data reasons. Mixing couples them and hides authz gaps. **But avoid ceremony:** no interfaces-for-one-implementation, no DI frameworks. Three folders is enough for most apps.

## Query optimization checklist

- Index every foreign key + every hot filter/sort (`EXPLAIN ANALYZE`).
- Select columns explicitly (DTOs), avoid `SELECT *` in hot paths.
- N+1: batch (`inArray`) or join in repository.
- Pagination: cursor (`WHERE id < cursor`) not `OFFSET` for big tables.

## Common Mistakes

**BAD:** `import.meta.env.DATABASE_URL` (baked at build! — v6+ inlines it).

**GOOD:** `process.env.DATABASE_URL` on the server (Module 33).

**BAD:** ORM entities serialized straight to the client (password hashes, internal flags).

**GOOD:** DTO mapping in services/actions.

**BAD:** transactions spanning external HTTP calls (Stripe inside DB transaction).

**GOOD:** transaction around DB only; external calls idempotent + retried (Module 36).

**BAD:** `text` columns for money.

**GOOD:** integer cents or `numeric`.

## Security Notes

- **Parameterized queries only** (Drizzle/`postgres-js` do this — never string-concatenate SQL). Module 23.
- Least-privilege DB roles; prod credentials never in dev machines.
- `onDelete: 'cascade'` is powerful — audit it; prefer soft-delete for user content.

## Performance Notes

- Pool sizing × serverless instances = connection ceiling; use a pooler (PgBouncer/Neon pooler) in prod.
- Prepared statements + module-level client = low latency.
- Cache hot read queries (Module 34) before scaling reads horizontally.

## Exercises

**Beginner.** Users + posts schema; `drizzle-kit generate`; seed script; list posts in a page (SSR).

**Intermediate.** Cursor pagination repository + service; write the `+1`-style next-cursor logic with tests.

**Production.** Transactional "transfer credits" with row locking (`FOR UPDATE`) and a concurrency test (two parallel transfers, no negative balance).

**Architecture Challenge.** The team wants Prisma "because we know it." When is Drizzle better, when is Prisma fine? Compare with: *both are valid; Drizzle = SQL control + edge-friendly + lighter; Prisma = richer tooling/migrations for teams that want it. Pick one; don't mix ORMs. The architecture (layering) matters more than the ORM.*

**Debugging Challenge.** Works locally, `FATAL: too many connections` in prod. Each request created `postgres()` client. Fix: module-level client + pooler; add a canary metric for pool wait time.

## MENTAL MODEL — Database Layer

> The database is a **shared, concurrent, slow-ish resource** behind a three-layer gate: route → service rules → repository SQL. Connections are hoarded and pooled; money is integers; ownership is enforced in queries. The ORM is a tool; the layering is the architecture.

## Official Documentation

- Astro DB removal guidance: https://docs.astro.build/en/guides/upgrade-to/v7/#removed-astrojsdb
- Drizzle: https://orm.drizzle.team/docs/overview
- node:sqlite: https://nodejs.org/api/sqlite.html

## What I Should Know Before Continuing

1. What replaced `@astrojs/db`? Where do migrations live now?
2. Draw the Action→Service→Repository→DB flow with authz at each hop.
3. Why are per-request DB clients a production outage waiting to happen?
4. Cursor vs offset pagination: which for infinite scroll and why?
