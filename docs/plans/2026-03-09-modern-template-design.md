# Modern Template Design

Date: 2026-03-09

## Purpose

A reusable Next.js starter template. Clone and go. No domain logic — only the infrastructure wiring that every project needs.

## Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Framework | Next.js 16 (App Router + Server Actions) | Frontend + backend in one, no tRPC needed |
| Language | TypeScript (strict) | Non-negotiable |
| ORM | Drizzle ORM | Fast, no binary, SQL-like, Edge-ready |
| Database | PostgreSQL on Neon | Serverless, free tier, no DevOps |
| Auth | Better-Auth | Email/password only by default, extensible |
| UI | shadcn/ui + Tailwind v4 | Own your components, no vendor lock-in |

## Folder Structure

```
src/
  app/
    (auth)/
      sign-in/page.tsx
      sign-up/page.tsx
    (dashboard)/
      dashboard/page.tsx
    api/
      auth/[...all]/route.ts
    layout.tsx
    page.tsx
  components/
    ui/                       # shadcn/ui components
  db/
    index.ts                  # Drizzle + Neon client
    schema.ts                 # Better-Auth required tables
  lib/
    auth.ts                   # Better-Auth server config
    auth-client.ts            # Better-Auth browser client
  middleware.ts
.env.example
drizzle.config.ts
```

## Data Layer

- `src/db/schema.ts` — Drizzle schema with Better-Auth required tables: `user`, `session`, `account`, `verification`
- `src/db/index.ts` — Neon serverless driver + Drizzle client exported as `db`
- `drizzle.config.ts` — points at `DATABASE_URL`, migrations output to `drizzle/`

## Auth Layer

- `src/lib/auth.ts` — Better-Auth server instance, email/password plugin, Drizzle adapter
- `src/lib/auth-client.ts` — `createAuthClient()` for client components
- `src/app/api/auth/[...all]/route.ts` — `{ GET, POST } = auth.handler`
- `src/middleware.ts` — protects `/dashboard/*`, redirects unauthenticated to `/sign-in`

## UI

shadcn/ui components installed: `button`, `input`, `label`, `card`, `form`.

Pages:
- `/sign-in` — email + password, Server Action
- `/sign-up` — email + password + confirm, Server Action
- `/dashboard` — protected, shows session user email + sign-out button

## Environment Variables

```
DATABASE_URL=postgresql://...
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=http://localhost:3000
```

## Intentionally Excluded

Add these per project as needed:
- Social OAuth providers
- 2FA / passkeys
- Organizations / teams
- Email sending (Resend, etc.)
- Domain-specific DB tables
- Deployment config (Vercel, Docker, etc.)
