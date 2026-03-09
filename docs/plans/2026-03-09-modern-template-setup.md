# Modern Template Setup Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Wire up a reusable Next.js 16 starter with Drizzle + Neon + Better-Auth (email/password) + shadcn/ui, ending with a working sign-in → dashboard flow.

**Architecture:** App Router with route groups `(auth)` and `(dashboard)`. Better-Auth handles sessions via a catch-all API route. Middleware protects dashboard routes. Drizzle talks to Neon over the serverless HTTP driver.

**Tech Stack:** Next.js 16, TypeScript strict, Drizzle ORM + drizzle-kit, Neon serverless, Better-Auth v1, shadcn/ui, Tailwind v4.

---

## Task 1: Install missing dependencies

**Files:**
- Modify: `package.json`

**Step 1: Install drizzle-kit (dev) and dotenv**

```bash
npm install -D drizzle-kit
npm install dotenv
```

Expected: `package.json` devDependencies gains `drizzle-kit`.

**Step 2: Verify**

```bash
npx drizzle-kit --version
```

Expected: prints a version like `0.xx.x`.

**Step 3: Commit**

```bash
git add package.json package-lock.json
git commit -m "chore: add drizzle-kit dev dependency"
```

---

## Task 2: Initialize shadcn/ui

**Files:**
- Create: `components.json`
- Modify: `src/app/globals.css`

**Step 1: Run shadcn init**

```bash
npx shadcn@latest init -d
```

When prompted, accept defaults. This creates `components.json` and updates `globals.css` with CSS variable tokens.

**Step 2: Verify**

```bash
ls components.json
```

Expected: file exists.

**Step 3: Commit**

```bash
git add components.json src/app/globals.css
git commit -m "chore: initialize shadcn/ui"
```

---

## Task 3: Install shadcn/ui components

**Files:**
- Create: `src/components/ui/button.tsx`, `input.tsx`, `label.tsx`, `card.tsx`

**Step 1: Add components**

```bash
npx shadcn@latest add button input label card
```

**Step 2: Verify files exist**

```bash
ls src/components/ui/
```

Expected: `button.tsx  card.tsx  input.tsx  label.tsx`

**Step 3: Commit**

```bash
git add src/components/
git commit -m "chore: add shadcn/ui base components"
```

---

## Task 4: Create .env.example and .env.local

**Files:**
- Create: `.env.example`
- Create: `.env.local` (fill in real values, gitignored)
- Modify: `.gitignore`

**Step 1: Create .env.example**

```
# .env.example
DATABASE_URL=postgresql://user:password@host/dbname?sslmode=require
BETTER_AUTH_SECRET=your-secret-here-min-32-chars
BETTER_AUTH_URL=http://localhost:3000
```

**Step 2: Create .env.local with real values**

Copy `.env.example` to `.env.local` and fill in:
- `DATABASE_URL` — your Neon connection string (from neon.tech dashboard → Connection string → choose "Node.js")
- `BETTER_AUTH_SECRET` — run `openssl rand -base64 32` to generate
- `BETTER_AUTH_URL` — `http://localhost:3000`

**Step 3: Ensure .env.local is gitignored**

Check `.gitignore` contains `.env.local` (Next.js adds this by default). If missing, add it.

**Step 4: Commit**

```bash
git add .env.example .gitignore
git commit -m "chore: add env example and gitignore"
```

---

## Task 5: Create Drizzle config

**Files:**
- Create: `drizzle.config.ts`

**Step 1: Write drizzle.config.ts**

```ts
import "dotenv/config";
import type { Config } from "drizzle-kit";

export default {
  schema: "./src/db/schema.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
} satisfies Config;
```

**Step 2: Commit**

```bash
git add drizzle.config.ts
git commit -m "chore: add drizzle config"
```

---

## Task 6: Create DB client and schema

**Files:**
- Create: `src/db/index.ts`
- Create: `src/db/schema.ts`

**Step 1: Write src/db/index.ts**

```ts
import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
import * as schema from "./schema";

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

**Step 2: Write src/db/schema.ts**

These are the exact tables Better-Auth requires for Postgres:

```ts
import { pgTable, text, timestamp, boolean } from "drizzle-orm/pg-core";

export const user = pgTable("user", {
  id: text("id").primaryKey(),
  name: text("name").notNull(),
  email: text("email").notNull().unique(),
  emailVerified: boolean("email_verified")
    .$defaultFn(() => false)
    .notNull(),
  image: text("image"),
  createdAt: timestamp("created_at")
    .$defaultFn(() => new Date())
    .notNull(),
  updatedAt: timestamp("updated_at")
    .$defaultFn(() => new Date())
    .notNull(),
});

export const session = pgTable("session", {
  id: text("id").primaryKey(),
  expiresAt: timestamp("expires_at").notNull(),
  token: text("token").notNull().unique(),
  createdAt: timestamp("created_at").notNull(),
  updatedAt: timestamp("updated_at").notNull(),
  ipAddress: text("ip_address"),
  userAgent: text("user_agent"),
  userId: text("user_id")
    .notNull()
    .references(() => user.id, { onDelete: "cascade" }),
});

export const account = pgTable("account", {
  id: text("id").primaryKey(),
  accountId: text("account_id").notNull(),
  providerId: text("provider_id").notNull(),
  userId: text("user_id")
    .notNull()
    .references(() => user.id, { onDelete: "cascade" }),
  accessToken: text("access_token"),
  refreshToken: text("refresh_token"),
  idToken: text("id_token"),
  accessTokenExpiresAt: timestamp("access_token_expires_at"),
  refreshTokenExpiresAt: timestamp("refresh_token_expires_at"),
  scope: text("scope"),
  password: text("password"),
  createdAt: timestamp("created_at").notNull(),
  updatedAt: timestamp("updated_at").notNull(),
});

export const verification = pgTable("verification", {
  id: text("id").primaryKey(),
  identifier: text("identifier").notNull(),
  value: text("value").notNull(),
  expiresAt: timestamp("expires_at").notNull(),
  createdAt: timestamp("created_at").$defaultFn(() => new Date()),
  updatedAt: timestamp("updated_at").$defaultFn(() => new Date()),
});
```

**Step 3: Commit**

```bash
git add src/db/
git commit -m "feat: add drizzle db client and better-auth schema"
```

---

## Task 7: Push schema to Neon

**Step 1: Run drizzle-kit push**

```bash
npx drizzle-kit push
```

Expected output: lists the 4 tables created (`user`, `session`, `account`, `verification`). No errors.

> Note: `push` applies the schema directly without generating migration files. Use `generate` + `migrate` for production migrations.

**Step 2: Verify in Neon console**

Open neon.tech → your project → Tables. You should see all 4 tables.

---

## Task 8: Configure Better-Auth server

**Files:**
- Create: `src/lib/auth.ts`

**Step 1: Write src/lib/auth.ts**

```ts
import { betterAuth } from "better-auth";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { db } from "@/db";
import * as schema from "@/db/schema";

export const auth = betterAuth({
  baseURL: process.env.BETTER_AUTH_URL,
  secret: process.env.BETTER_AUTH_SECRET,
  database: drizzleAdapter(db, {
    provider: "pg",
    schema,
  }),
  emailAndPassword: {
    enabled: true,
  },
});
```

**Step 2: Commit**

```bash
git add src/lib/auth.ts
git commit -m "feat: configure better-auth server"
```

---

## Task 9: Configure Better-Auth client

**Files:**
- Create: `src/lib/auth-client.ts`

**Step 1: Write src/lib/auth-client.ts**

```ts
import { createAuthClient } from "better-auth/react";

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_BETTER_AUTH_URL,
});
```

**Step 2: Add NEXT_PUBLIC_BETTER_AUTH_URL to .env.local and .env.example**

Add this line:
```
NEXT_PUBLIC_BETTER_AUTH_URL=http://localhost:3000
```

**Step 3: Commit**

```bash
git add src/lib/auth-client.ts .env.example
git commit -m "feat: add better-auth browser client"
```

---

## Task 10: Add Better-Auth API route

**Files:**
- Create: `src/app/api/auth/[...all]/route.ts`

**Step 1: Write route.ts**

```ts
import { auth } from "@/lib/auth";
import { toNextJsHandler } from "better-auth/next-js";

export const { GET, POST } = toNextJsHandler(auth);
```

**Step 2: Commit**

```bash
git add src/app/api/
git commit -m "feat: add better-auth api route handler"
```

---

## Task 11: Add middleware to protect routes

**Files:**
- Create: `src/middleware.ts`

**Step 1: Write src/middleware.ts**

```ts
import { NextRequest, NextResponse } from "next/server";
import { getSessionCookie } from "better-auth/cookies";

export async function middleware(request: NextRequest) {
  const sessionCookie = getSessionCookie(request);

  if (!sessionCookie) {
    return NextResponse.redirect(new URL("/sign-in", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*"],
};
```

**Step 2: Commit**

```bash
git add src/middleware.ts
git commit -m "feat: add auth middleware for dashboard routes"
```

---

## Task 12: Create sign-in page

**Files:**
- Create: `src/app/(auth)/sign-in/page.tsx`

**Step 1: Write sign-in page**

```tsx
"use client";

import { useState } from "react";
import { authClient } from "@/lib/auth-client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import Link from "next/link";

export default function SignInPage() {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError("");

    const { error } = await authClient.signIn.email({
      email,
      password,
      callbackURL: "/dashboard",
    });

    if (error) {
      setError(error.message ?? "Sign in failed");
      setLoading(false);
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center">
      <Card className="w-full max-w-sm">
        <CardHeader>
          <CardTitle>Sign in</CardTitle>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit} className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="email">Email</Label>
              <Input
                id="email"
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                required
              />
            </div>
            <div className="space-y-2">
              <Label htmlFor="password">Password</Label>
              <Input
                id="password"
                type="password"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                required
              />
            </div>
            {error && <p className="text-sm text-red-500">{error}</p>}
            <Button type="submit" className="w-full" disabled={loading}>
              {loading ? "Signing in..." : "Sign in"}
            </Button>
            <p className="text-center text-sm text-muted-foreground">
              No account?{" "}
              <Link href="/sign-up" className="underline">
                Sign up
              </Link>
            </p>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}
```

**Step 2: Commit**

```bash
git add src/app/
git commit -m "feat: add sign-in page"
```

---

## Task 13: Create sign-up page

**Files:**
- Create: `src/app/(auth)/sign-up/page.tsx`

**Step 1: Write sign-up page**

```tsx
"use client";

import { useState } from "react";
import { authClient } from "@/lib/auth-client";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import Link from "next/link";

export default function SignUpPage() {
  const [name, setName] = useState("");
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setLoading(true);
    setError("");

    const { error } = await authClient.signUp.email({
      name,
      email,
      password,
      callbackURL: "/dashboard",
    });

    if (error) {
      setError(error.message ?? "Sign up failed");
      setLoading(false);
    }
  }

  return (
    <div className="min-h-screen flex items-center justify-center">
      <Card className="w-full max-w-sm">
        <CardHeader>
          <CardTitle>Create account</CardTitle>
        </CardHeader>
        <CardContent>
          <form onSubmit={handleSubmit} className="space-y-4">
            <div className="space-y-2">
              <Label htmlFor="name">Name</Label>
              <Input
                id="name"
                type="text"
                value={name}
                onChange={(e) => setName(e.target.value)}
                required
              />
            </div>
            <div className="space-y-2">
              <Label htmlFor="email">Email</Label>
              <Input
                id="email"
                type="email"
                value={email}
                onChange={(e) => setEmail(e.target.value)}
                required
              />
            </div>
            <div className="space-y-2">
              <Label htmlFor="password">Password</Label>
              <Input
                id="password"
                type="password"
                value={password}
                onChange={(e) => setPassword(e.target.value)}
                required
                minLength={8}
              />
            </div>
            {error && <p className="text-sm text-red-500">{error}</p>}
            <Button type="submit" className="w-full" disabled={loading}>
              {loading ? "Creating account..." : "Create account"}
            </Button>
            <p className="text-center text-sm text-muted-foreground">
              Have an account?{" "}
              <Link href="/sign-in" className="underline">
                Sign in
              </Link>
            </p>
          </form>
        </CardContent>
      </Card>
    </div>
  );
}
```

**Step 2: Commit**

```bash
git add src/app/
git commit -m "feat: add sign-up page"
```

---

## Task 14: Create sign-out button component

**Files:**
- Create: `src/components/sign-out-button.tsx`

**Step 1: Write the component**

```tsx
"use client";

import { authClient } from "@/lib/auth-client";
import { useRouter } from "next/navigation";
import { Button } from "@/components/ui/button";

export function SignOutButton() {
  const router = useRouter();

  async function handleSignOut() {
    await authClient.signOut({
      fetchOptions: {
        onSuccess: () => router.push("/sign-in"),
      },
    });
  }

  return (
    <Button variant="outline" onClick={handleSignOut}>
      Sign out
    </Button>
  );
}
```

**Step 2: Commit**

```bash
git add src/components/sign-out-button.tsx
git commit -m "feat: add sign-out button component"
```

---

## Task 15: Create dashboard page

**Files:**
- Create: `src/app/(dashboard)/dashboard/page.tsx`

**Step 1: Write dashboard page**

```tsx
import { auth } from "@/lib/auth";
import { headers } from "next/headers";
import { redirect } from "next/navigation";
import { SignOutButton } from "@/components/sign-out-button";

export default async function DashboardPage() {
  const session = await auth.api.getSession({
    headers: await headers(),
  });

  if (!session) {
    redirect("/sign-in");
  }

  return (
    <div className="p-8">
      <div className="flex items-center justify-between mb-8">
        <h1 className="text-2xl font-bold">Dashboard</h1>
        <SignOutButton />
      </div>
      <p className="text-muted-foreground">
        Signed in as <span className="font-medium text-foreground">{session.user.email}</span>
      </p>
    </div>
  );
}
```

**Step 2: Commit**

```bash
git add src/app/
git commit -m "feat: add dashboard page"
```

---

## Task 16: Update root layout and landing page

**Files:**
- Modify: `src/app/layout.tsx`
- Modify: `src/app/page.tsx`

**Step 1: Update src/app/layout.tsx**

Replace the file with:

```tsx
import type { Metadata } from "next";
import { Geist, Geist_Mono } from "next/font/google";
import "./globals.css";

const geistSans = Geist({
  variable: "--font-geist-sans",
  subsets: ["latin"],
});

const geistMono = Geist_Mono({
  variable: "--font-geist-mono",
  subsets: ["latin"],
});

export const metadata: Metadata = {
  title: "Modern Template",
  description: "Next.js + Drizzle + Better-Auth + Neon starter",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body className={`${geistSans.variable} ${geistMono.variable} antialiased`}>
        {children}
      </body>
    </html>
  );
}
```

**Step 2: Update src/app/page.tsx**

```tsx
import { redirect } from "next/navigation";

export default function RootPage() {
  redirect("/sign-in");
}
```

**Step 3: Commit**

```bash
git add src/app/layout.tsx src/app/page.tsx
git commit -m "feat: update root layout and redirect landing to sign-in"
```

---

## Task 17: Final verification

**Step 1: Run dev server**

```bash
npm run dev
```

Expected: no TypeScript errors, server starts on port 3000.

**Step 2: Test the full flow**

1. Open `http://localhost:3000` — should redirect to `/sign-in`
2. Go to `/sign-up` — create an account
3. After sign-up, should redirect to `/dashboard`
4. Dashboard should show your email
5. Sign out — should redirect to `/sign-in`
6. Try accessing `/dashboard` without signing in — should redirect to `/sign-in`

**Step 3: Run build**

```bash
npm run build
```

Expected: build succeeds with no errors.

**Step 4: Commit**

If you had to fix anything during testing, commit those fixes. Then tag the template:

```bash
git tag v1.0.0
```
