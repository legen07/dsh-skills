---
name: nextjs-16-edge
description: |
  Production-grade Next.js 16+ skill with Bun, Turbopack, Biome, and Cloudflare Edge deployment.
  Enforces the strict 9-layer architecture, WAAPI animations (no external animation libraries),
  and modular CSS (no Tailwind). Fully static site with Edge function requests via @cloudflare/workbox.
  Secrets stored in .dev.vars (dev) → synced to Cloudflare Secrets (deploy).
  AI-generic — no framework or AI-specific bindings.
license: MIT
metadata:
  author: Joe Legen <https://github.com/legen07>
  version: 1.0.0
  domain: frontend
  runtime: bun + node
  triggers:
    - nextjs 16
    - next.js 16
    - cloudflare
    - wrangler
    - bun
    - turbopack
    - biome
    - 9-layer
    - nextjs-edge
    - wrangler dev
    - cloudflare d1

---

# Next.js 16+ Edge Architecture

Build production-grade, fully static Next.js 16+ applications deployed to Cloudflare Pages/Workers
with Edge function requests. Uses the strict 9-layer architecture, Bun for execution, Turbopack
for bundling, Biome for linting/type-checking, WAAPI for animations, and modular CSS.

---

## 1. Project Structure (9-Layer Architecture)

```
project/
├── prisma/
│   └── schema.prisma                    # Layer 1: D1 schema via Drizzle/Prisma adapter
├── db/
│   └── index.ts                          # Layer 2: Database client singleton
├── repository/
│   ├── user.repository.ts                # Layer 3: all DB queries (pure access layer)
│   └── post.repository.ts
├── services/
│   ├── user.service.ts                   # Layer 4: business logic, validation, side effects
│   └── post.service.ts
├── actions/
│   ├── user.actions.ts                   # Layer 7: server actions (client → server bridge)
│   └── post.actions.ts
├── hooks/
│   ├── useUser.ts                        # Layer 8: client state management
│   └── usePosts.ts
├── middleware.ts                         # Layer 5: auth, redirects, rate limits (Edge)
├── app/
│   ├── layout.tsx                        # Root layout: html/body required
│   ├── loading.tsx                       # Global loading fallback
│   ├── error.tsx                         # Error boundary ('use client')
│   ├── not-found.tsx                     # 404 page
│   ├── (marketing)/
│   │   └── page.tsx                      # Route group
│   ├── (dashboard)/
│   │   ├── layout.tsx                    # Shared dashboard layout
│   │   └── page.tsx                      # Dashboard route
│   ├── api/
│   │   └── [resource]/route.ts          # Layer 6: Route Handlers (Edge-compatible)
│   └── components/
│       └── ui/
│           ├── Button.tsx
│           └── AnimatedCard.tsx          # WAAPI animations only
├── public/
├── wrangler.toml                         # Wrangler config for Cloudflare dev/deploy
├── next.config.ts                        # Bun + Turbopack config
├── biome.json                            # Biome config (replaces ESLint)
├── tsconfig.json
├── package.json
└── .dev.vars                           # Secrets for development (git-ignored)
```

**Layer Dependency Rule:** Each layer may only import from the layer directly below it. Never skip layers.

---

## 2. Runtime & Build Tools

### Bun (execution runtime)
```json
// package.json
{
  "engines": {
    "bun": "^1.40.0"
  },
  "scripts": {
    "dev": "bun run next dev",
    "build": "bun run next build",
    "preview": "bun run next preview",
    "lint": "bun run biome check .",
    "typecheck": "bun run biome check . --ts",
    "wrangler-dev": "bun run wrangler dev .",
    "deploy": "bun run wrangler deploy .",
    "sync-secrets": "bun run sync-secrets.js"
  }
}
```

### Turbopack (bundling)
Turbopack is the default for `next dev` in Next.js 16. Enable explicitly:

```ts
// next.config.ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  experimental: {
    turbopack: true,          // Force Turbopack
    reactCompiler: true,      // React Compiler (Next.js 16)
    cacheComponents: true,    // Cache Components (Next.js 16)
    runtime: 'edge'           // Edge runtime for App Router
  },
  // ...
}

export default config
```

### Biome (linting + type-checking)
Biome replaces ESLint + Prettier + other tools.

```json
// biome.json
{
  "extends": "recommended",
  "plugins": ["jsx", "ts", "turbo"],
  "settings": {
    "ts": { "target": "ES2022" },
    "jsx": { "runtime": "react-19" }
  },
  "rules": {
    "use-define-over-constant": "warn",
    "no-throw-literal": "warn",
    "semi": "warn",
    "prefer-nullish-coalescing": "error",
    "return-from-function-with-assignment": "error"
  }
}
```

Run: `bun run biome check . --ts`

---

## 3. Cloudflare Wrangler (Dev + Deploy)

### Wrangler Config
```toml
# wrangler.toml
[[dev]]
runtime = "cloudflare"
port = 3000

[deploy]
name = "my-nextapp"
environment = "prod"

[[static_files]]
src = "./out"
dest = "/"

[[static_files.includes]]
src = "**/*.{html,js,css,jsx,tsx}"
```

### Dev Mode
```bash
# Wrangler dev starts the Edge Runtime locally, serving your Next.js build
bun run wrangler dev .
```

### Deploy Mode
```bash
# Sync .dev.vars to Cloudflare Secrets, then deploy
bun run sync-secrets.js
bun run wrangler deploy .
```

---

## 4. Secrets Management

### Development (.dev.vars)
```env
# .dev.vars — never commit, add to .gitignore
DATABASE_URL="sqlite:./data.d1.sqlite"
# Or for D1 test: DATABASE_URL="D1:your-d1-id:dev"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

### Development (D1 SQLite)
```ts
// .dev.vars
DATABASE_URL="sqlite:./data.d1.sqlite"
```

### Production (D1 via Cloudflare)
```ts
// .dev.vars
DATABASE_URL="D1:your-d1-id:prod"
```

### Sync Script
```ts
// sync-secrets.js
import { CloudflareWorkers } from '@cloudflare/worker-remote/lib/worker-remote';
import { fileURLToPath } from 'url';
import { join } from 'path';
import fs from 'fs';

const path = fileURLToPath(import.meta.url);
const dir = join(path, '../');

// Read .dev.vars
const envContent = fs.readFileSync(join(dir, '.dev.vars'), 'utf-8');
const vars = new Map();
envContent
  .split('\n')
  .forEach((line) => {
    const [key, value] = line.split('=');
    if (key && value && !key.startsWith('#')) vars.set(key, value);
  });

console.log('Current .dev.vars:');
console.log([...vars.entries()]);
```

**Deployment secret flow:**
1. Add secrets to `.dev.vars`
2. Run `wrangler secret set DATABASE_URL --from-file .dev.vars`
3. Run `wrangler deploy`

---

## 5. Data Layer (Layers 1-3)

### Layer 1: Schema (Drizzle + D1)
```prisma
// prisma/schema.prisma
generator client {
  provider = "drizzle-orm"
}

datasource ds {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

For D1 specifically (Drizzle D1 adapter):

```ts
// drizzle.config.ts
import { defineConfig } from 'drizzle-kit';
import ts from 'typescript';

export default defineConfig({
  schema: './prisma/schema.prisma',
  out: './drizzle',
  dialect: 'sqlite' // D1 uses SQLite
});
```

### Layer 2: Database Client Singleton
```ts
// db/index.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import sqlite from 'drizzle-orm/sqlite-core';

export const db = db;
```

### Layer 3: Repository
```ts
// repository/user.repository.ts
import { db } from '@/db';
import { select, eq } from 'drizzle-orm';
import { users } from '@/drizzle';

export const userRepository = {
  findById: (id: string) =>
    db.select().from(users).where(eq(users.id, id)).limit(1),

  findByEmail: (email: string) =>
    db.select().from(users).where(eq(users.email, email)).limit(1),

  findMany: () =>
    db.select().from(users),

  create: (data: { email: string; name: string }) =>
    db.insert(users).values(data).returning(),

  update: (id: string, data: Partial<{ email: string; name: string }>) =>
    db.update(users).set(data).where(eq(users.id, id)).returning(),

  delete: (id: string) =>
    db.delete(users).where(eq(users.id, id))
};
```

---

## 6. Business Logic (Layer 4)

```ts
// services/user.service.ts
import { userRepository } from '@/repository/user.repository';
import { z } from 'zod';

export class UserNotFoundError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'UserNotFoundError';
  }
}

export class EmailTakenError extends Error {
  constructor() {
    super('Email is already registered');
    this.name = 'EmailTakenError';
  }
}

const RegisterSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});

export const userService = {
  async register(email: string, name: string, passwordHash: string) {
    const existing = await userRepository.findByEmail(email);
    if (existing?.length > 0) throw new EmailTakenError();

    return userRepository.create({ email, name });
  },

  async getUser(id: string) {
    const [user] = await userRepository.findById(id);
    if (!user) throw new UserNotFoundError(id);
    return user;
  }
};
```

---

## 7. Server Actions (Layer 7)

```ts
// actions/user.actions.ts
'use server'

import { cacheRevalidate } from 'next/cache';
import { userService, EmailTakenError } from '@/services/user.service';
import { RegisterSchema } from '@/services/user.service';

export type ActionResult<T> =
  | { success: true; data: T }
  | { success: false; error: string };

export async function registerUser(formData: FormData): Promise<ActionResult<{ id: string }>> {
  const parsed = RegisterSchema.safeParse(Object.fromEntries(formData));
  if (!parsed.success) {
    return { success: false, error: parsed.error.errors[0].message };
  }

  try {
    const passwordHash = await hashPassword(parsed.data.password);
    const user = await userService.register(parsed.data.email, parsed.data.name, passwordHash);
    cacheRevalidate('/dashboard');
    return { success: true, data: { id: user.id } };
  } catch (err) {
    if (err instanceof EmailTakenError) return { success: false, error: err.message };
    return { success: false, error: 'Registration failed. Please try again.' };
  }
}
```

---

## 8. Hooks (Layer 8)

```ts
// hooks/useUpdateProfile.ts
'use client'

import { useTransition } from 'react';
import { updateProfile } from '@/actions/user.actions';

export function useUpdateProfile() {
  const [isPending, startTransition] = useTransition();

  const update = async (formData: FormData) => {
    startTransition(async () => {
      const result = await updateProfile(formData);
      if (!result.success) console.error(result.error);
    });
  };

  return { update, isPending };
}
```

---

## 9. UI Components with WAAPI (Layer 9)

**No external animation libraries.** Use Web Animations API only.

```tsx
// app/components/ui/AnimatedCard.tsx
'use client'

import { useRef, useEffect } from 'react';
import { styleModule } from './animated-card.css.module';

interface AnimatedCardProps {
  children: React.ReactNode;
  animateOnMount?: boolean;
}

export function AnimatedCard({ children, animateOnMount = false }: AnimatedCardProps) {
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!ref.current || !animateOnMount) return;

    const animation = ref.current.createAnimation(
      [
        { transform: 'scale(0.95)', opacity: '0' },
        { transform: 'scale(1)', opacity: '1' },
      ],
      { duration: 300, easing: 'ease-out' }
    );
    animation.play();
  }, [animateOnMount]);

  return <div ref={ref} className={styleModule.card}>{children}</div>;
}
```

### WAAPI Utility Functions
```ts
// lib/animations.ts
export function createFadeIn(element: HTMLElement) {
  return element.createAnimation(
    [{ opacity: '0' }, { opacity: '1' }],
    { duration: 500, easing: 'ease-out' }
  );
}

export function createSlideInLeft(element: HTMLElement, delay: number = 0) {
  return element.createAnimation(
    [{ transform: 'translateX(-20px)', opacity: '0' }, { transform: 'translateX(0)', opacity: '1' }],
    { duration: 400, easing: 'ease-out', delay }
  );
}

export function createStaggerChildren(parent: HTMLElement, delay: number = 100) {
  return Array.from(parent.children).map((child, i) =>
    createSlideInLeft(child as HTMLElement, i * delay)
  );
}
```

### Usage in a page:
```tsx
// app/page.tsx
import { createFadeIn } from '@/lib/animations';
import AnimatedCard from '@/components/ui/AnimatedCard';
import { useEffect } from 'react';

export default function Home() {
  useEffect(() => {
    const el = document.querySelector('#hero');
    if (el) createFadeIn(el).play();
  }, []);

  return (
    <main>
      <section id="hero" className="text-center py-20">
        <h1>Hello World</h1>
      </section>
      <AnimatedCard animateOnMount>
        <p>Welcome to the edge.</p>
      </AnimatedCard>
    </main>
  );
}
```

---

## 10. Middleware (Layer 5 — Edge)

```ts
// middleware.ts
import { NextResponse, type NextRequest } from 'next/server';

export function middleware(request: NextRequest): NextResponse {
  const token = request.cookies.get('auth_token');

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*'],
};
```

---

## 11. Cache Components (Next.js 16+)

```ts
// pages/users.tsx
import { cacheLife, cacheTag } from 'next/cache';

export default async function UsersPage() {
  'use cache';
  cacheLife('hours');
  cacheTag('users');

  // Fetch fresh user data, revalidate via updateTag from mutations
  const res = await fetch('https://api.example.com/users');
  return <ul>{(await res.json()).map((u: any) => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

From a Server Action after mutation:
```ts
import { updateTag } from 'next/cache';

export async function deleteUser(id: string) {
  await db.user.delete({ where: { id } });
  updateTag('users'); // Invalidate cache
}
```

---

## 12. Route Handlers (Layer 6 — Edge-Compatible)

```ts
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

export async function GET() {
  // Edge runtime: use only Edge-compatible APIs
  const res = await fetch('https://api.example.com/users', {
    cache: 'no-store',
  });
  return NextResponse.json(await res.json());
}

export async function POST(request: NextRequest) {
  const body = await request.json();
  // ... handle creation
  return NextResponse.json({ message: 'created' }, { status: 201 });
}
```

---

## 13. Build Commands

```bash
# Dev server (Turbopack, Bun)
bun run next dev

# Production build
bun run next build

# Preview build
bun run next preview

# Lint + typecheck (Biome)
bun run biome check . --ts

# Wrangler dev (Edge runtime locally)
bun run wrangler dev .

# Deploy to Cloudflare
bun run wrangler deploy .
```

---

## 14. Implementation Checklist

- [ ] Use Bun as runtime (`bun run next dev`)
- [ ] Use Turbopack (enabled by default, but `experimental: { turbopack: true }`)
- [ ] Use Biome for linting (`biome check . --ts`)
- [ ] Follow 9-layer architecture (never skip layers)
- [ ] Use Edge runtime for middleware and API routes
- [ ] Use Wrangler for dev and deployment
- [ ] Store secrets in `.dev.vars`, sync to Cloudflare Secrets
- [ ] Use D1 for database (SQLite adapter via Drizzle)
- [ ] Use WAAPI for animations (no external animation libraries)
- [ ] Use modular CSS (CSS modules or styled-components)
- [ ] Use Next.js 16+ features: Cache Components, React Compiler, Turbopack
- [ ] Use `'use cache'` directive for explicit caching
- [ ] Use `updateTag()` for cache invalidation on mutations
- [ ] Use Server Actions for mutations, not API routes (when possible)
- [ ] Keep Server Components by default, add `'use client'` only for interactivity
- [ ] Use Zod for validation in Server Actions
- [ ] Use proper error handling: typed errors + ActionResult pattern
- [ ] Test cache behavior with curl or Next.js test client

---

## 15. Anti-Patterns

| ❌ Don't | ✅ Do |
|----------|-------|
| Use external animation libraries (Framer Motion, GSAP) | Use WAAPI only |
| Use Tailwind CSS | Use modular CSS (CSS modules, styled-components, or vanilla) |
| Use ESLint | Use Biome |
| Use Webpack | Use Turbopack |
| Skip layers in the 9-layer architecture | Follow strict layer boundaries |
| Store secrets in environment variables without Cloudflare sync | Use `.dev.vars` → Cloudflare Secrets |
| Fetch in Client Components | Fetch in Server Components |
| Throw raw errors from Server Actions | Return `ActionResult` with typed errors |
| Use `getServerSideProps` / `getStaticProps` | Use Server Components + `generateStaticParams` |
| Assume Node.js APIs in Edge routes | Use Edge-compatible APIs only |

---

## 16. AI-Generic

This skill has no dependency on any specific AI platform, model, or provider. It covers standard Next.js 16+ patterns that work independently of any AI integration. Any AI integration (RAG, tools, function calling) should be built on top of these layers, typically in **Server Actions** or **Route Handlers**.