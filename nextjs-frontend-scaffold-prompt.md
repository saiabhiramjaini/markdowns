# Next.js Frontend-Only Scaffold Prompt

Copy-paste this entire prompt to scaffold a production-ready **frontend-only** Next.js app from scratch.
Replace `{{APP_NAME}}` with your app name (kebab-case, e.g. `my-app`).

**Use this when:** the app has no database, no server-side mutations, no auth on the server. The frontend may call an external API you don't control (Stripe public endpoints, public REST APIs, your own backend on a different service), or it may be purely static (marketing, docs, landing pages).

**Use the full-stack prompt instead when:** you need a database, NextAuth + Google OAuth, server actions, audit trails, or any first-party backend logic.

---

## PROMPT START

Scaffold a production-ready frontend-only Next.js app called `{{APP_NAME}}`. Follow every instruction exactly — don't add extras, don't skip steps.

---

### 1. Init

> **Bun is the package manager and script runner — Next.js still runs on Node.js.**
> `bun install`, `bun add`, `bunx` are all Bun. But `next dev` / `next build` / `next start` execute under Node.js internally. Bun's value here is fast installs, fast `bunx`, and Vitest integration. The app runtime is Node.js.

```bash
bun create next-app {{APP_NAME}} --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --no-git
cd {{APP_NAME}}
git init
bun add -d husky lint-staged prettier
bunx husky init
```

---

### 2. Package versions

Set these exact deps in `package.json` (adjust patch versions to latest at install time):

**Runtime deps:**
```json
{
  "next": "^16",
  "react": "^19",
  "react-dom": "^19",
  "zod": "^4",
  "pino": "^9",
  "clsx": "^2",
  "tailwind-merge": "^3",
  "react-hook-form": "^7",
  "@hookform/resolvers": "^5",
  "@tanstack/react-query": "^5",
  "@tanstack/react-query-devtools": "^5",
  "zustand": "^5"
}
```

**Dev deps:**
```json
{
  "vitest": "^3",
  "@vitejs/plugin-react": "^4",
  "@testing-library/react": "^16",
  "@testing-library/jest-dom": "^6",
  "jsdom": "^25",
  "pino-pretty": "^11",
  "typescript": "^5.9",
  "husky": "^9",
  "lint-staged": "^15",
  "prettier": "^3"
}
```

Add to root `package.json`:
```json
{
  "lint-staged": {
    "*.{ts,tsx,js,jsx,mjs,cjs,json,yml,yaml,md,css}": "prettier --write"
  },
  "overrides": {
    "esbuild": ">=0.25.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

---

### 3. shadcn/ui + theme

**Step 1 — Init shadcn:**
```bash
bunx shadcn@latest init
```

When prompted:
- Style: **Default**
- Base color: **Slate**
- CSS variables: **Yes**

**Step 2 — Add the base component set:**
```bash
bunx shadcn@latest add button input label textarea dialog dropdown-menu badge toast card separator skeleton tabs select checkbox radio-group switch tooltip popover sheet command avatar form
```

**Step 3 — Get `globals.css` from tweakcn.com:**

Go to [tweakcn.com](https://tweakcn.com) → pick a theme → copy the generated CSS → replace the contents of `src/app/globals.css` entirely with what tweakcn gives you.

**Rules:**
- Always install via `bunx shadcn@latest add <name>` — never copy-paste manually
- Never duplicate a shadcn component — extend via `cva` variants
- Compose complex UI by combining primitives — don't rewrite them

---

### 4. Prettier config

Create `.prettierrc`:
```json
{
  "semi": true,
  "singleQuote": false,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "plugins": []
}
```

Create `.prettierignore`:
```
node_modules/
.next/
out/
build/
dist/
*.lock
**/next-env.d.ts
**/*.tsbuildinfo
```

---

### 5. Husky hooks

**`.husky/commit-msg`:**
```sh
#!/bin/sh
commit_msg=$(cat "$1")
if ! echo "$commit_msg" | grep -qE "^(feat|fix|docs|style|refactor|test|chore|perf): .+"; then
  echo "❌ Invalid commit message format!"
  echo "Format: type: description"
  echo "Types: feat, fix, docs, style, refactor, test, chore, perf"
  exit 1
fi
```

**`.husky/pre-commit`:**
```sh
#!/bin/sh
export PATH="$HOME/.bun/bin:$PATH"
set -e
echo "💅 Formatting staged files..."
bunx lint-staged
echo "🔍 Linting..."
bunx next lint
echo "✅ pre-commit passed"
```

**`.husky/pre-push`:**
```sh
#!/bin/sh
export PATH="$HOME/.bun/bin:$PATH"
set -e
echo "🔍 Linting..."
bunx next lint
echo "💅 Checking formatting..."
bunx prettier --check "**/*.{ts,tsx,js,jsx,json,md,css}"
echo "🔎 Type checking..."
bunx tsc --noEmit
echo "🧪 Running tests..."
bunx vitest run
echo "🔒 Auditing dependencies..."
bun audit --audit-level=high
echo "✅ All checks passed — safe to push"
```

Make all hooks executable:
```bash
chmod +x .husky/commit-msg .husky/pre-commit .husky/pre-push
```

---

### 6. Core lib files

**`src/lib/utils.ts`** — the `cn()` helper shadcn already installed. Verify it exists:
```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**`src/lib/log.ts`** — used for server-side rendering errors (server components, route handlers, instrumentation):
```typescript
import pino, { type Logger, type LoggerOptions } from "pino";

const isProd = process.env.NODE_ENV === "production";
const isTest = process.env.NODE_ENV === "test";

const PINO_TO_SEVERITY: Record<string, string> = {
  trace: "DEBUG", debug: "DEBUG", info: "INFO",
  warn: "WARNING", error: "ERROR", fatal: "CRITICAL",
};

const baseOptions: LoggerOptions = {
  level: process.env.LOG_LEVEL ?? (isProd ? "info" : "debug"),
  serializers: { err: pino.stdSerializers.err, error: pino.stdSerializers.err },
  formatters: {
    level(label) {
      return { level: label, severity: PINO_TO_SEVERITY[label] ?? label.toUpperCase() };
    },
    bindings() {
      return { service: "{{APP_NAME}}" };
    },
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: ["password", "*.password", "token", "*.token", "apiKey", "*.apiKey", "authorization", "*.authorization"],
    censor: "[REDACTED]",
  },
};

const devTransport: LoggerOptions["transport"] =
  !isProd && !isTest
    ? { target: "pino-pretty", options: { colorize: true, translateTime: "SYS:HH:MM:ss.l", ignore: "pid,hostname,service,severity" } }
    : undefined;

export const log: Logger = pino({ ...baseOptions, ...(devTransport ? { transport: devTransport } : {}) });
```

**`src/lib/report-client-error.ts`** — for client-component errors:
```typescript
export function reportClientError(err: unknown, context?: Record<string, unknown>): void {
  const message = err instanceof Error ? err.message : String(err);
  const stack = err instanceof Error ? err.stack : undefined;
  fetch("/api/log/client-error", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ message, stack, context }),
  }).catch(() => {});
}
```

**`src/app/api/log/client-error/route.ts`** — receives client errors and forwards to your log backend (Cloud Run / Vercel logs / wherever pino's JSON ships):
```typescript
import { NextRequest, NextResponse } from "next/server";
import { log } from "@/lib/log";

export const runtime = "nodejs";

export async function POST(req: NextRequest): Promise<NextResponse> {
  try {
    const body = await req.json();
    log.error({ ...body, source: "client" }, "client.error");
  } catch {}
  return NextResponse.json({ ok: true });
}
```

---

### 7. Zod conventions

Zod is the **only** validation library. Use it at every system boundary — form input, external API responses, env vars.

```typescript
// ✅ Always safeParse — never .parse() which throws
const parsed = CreateThingSchema.safeParse(input);
if (!parsed.success) { /* handle */ }

// ✅ Every string field gets .trim() + .min(1) + .max() with a custom user-facing message
const NameSchema = z
  .string({ message: "Name is required" })
  .trim()
  .min(1, { message: "Name is required" })
  .max(120, { message: "Name must be 120 characters or fewer" });

// ✅ Always export inferred types alongside schemas
export const ContactFormSchema = z.object({
  name: NameSchema,
  email: z.string().email({ message: "Enter a valid email" }),
  message: z.string().min(1, { message: "Message is required" }).max(2000),
});
export type ContactFormInput = z.infer<typeof ContactFormSchema>;
```

**Error message rules:**
- Messages are **user-facing** — no TypeScript jargon
- Required field: `"Name is required"` not `"name must be a string"`
- Length limit: `"Name must be 120 characters or fewer"` not `"String too long"`

**Where Zod runs:**

| Boundary | Use Zod? |
|---|---|
| Form input (`react-hook-form` + `zodResolver`) | ✅ Always |
| External API response | ✅ Always (you don't control the shape) |
| Env var validation | ✅ At first use |
| Internal function calls | ❌ Trust TypeScript |

---

### 8. Forms — react-hook-form + Zod (mandatory pattern)

**Every form uses `react-hook-form` + `zodResolver`.** No exceptions. Hand-rolled `useState` form state is banned.

```tsx
// src/features/contact/components/contact-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { toast } from "sonner";
import { ContactFormSchema, type ContactFormInput } from "@/features/contact/schema";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from "@/components/ui/form";

export function ContactForm() {
  const form = useForm<ContactFormInput>({
    resolver: zodResolver(ContactFormSchema),
    defaultValues: { name: "", email: "", message: "" },
  });

  async function onSubmit(values: ContactFormInput) {
    const res = await fetch("/api/contact", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(values),
    });

    if (!res.ok) {
      const error = await res.json().catch(() => ({}));
      form.setError("root", { message: error?.error ?? "Something went wrong" });
      return;
    }

    toast.success("Message sent");
    form.reset();
  }

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Name</FormLabel>
              <FormControl><Input {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl><Input type="email" {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <FormField
          control={form.control}
          name="message"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Message</FormLabel>
              <FormControl><Textarea {...field} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {form.formState.errors.root && (
          <p className="text-sm text-destructive">{form.formState.errors.root.message}</p>
        )}

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Sending..." : "Send"}
        </Button>
      </form>
    </Form>
  );
}
```

**Rules:**
- **One schema, two consumers.** The Zod schema lives in `feature/schema.ts`. The form imports it for `zodResolver`. The API response handler (server-side or external API) validates with the same schema.
- **Always `zodResolver`.** Never hand-roll validation.
- **Default values must cover every field.** Skipping triggers "uncontrolled to controlled" warnings.
- **Server-side error → `form.setError("root", ...)`**. Field errors render automatically via `<FormMessage />`.
- **Disable submit while pending.** `disabled={form.formState.isSubmitting}` + a pending label.
- **For optional nullable fields**, controlled inputs need `value={field.value ?? ""}` — React doesn't accept `null`.

---

### 9. State management — React Query + Zustand

Two different concerns, two different tools:

| State type | Tool | Examples |
|---|---|---|
| **Server state** (data fetched from an API) | React Query | User profile, product list, posts, search results |
| **Client state** (UI-only, lives on the client) | Zustand | Sidebar open/closed, theme override, filters not yet submitted |

Never use `useState` + `useEffect` to fetch data — that path leads to race conditions, duplicate fetches, and stale data. Always React Query.

**React Query provider** — wrap the app in the root layout's client provider:

```tsx
// src/components/providers.tsx
"use client";

import { useState } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

export function Providers({ children }: { children: React.ReactNode }) {
  const [client] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute — tune per query if needed
            refetchOnWindowFocus: false,
            retry: 1,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={client}>
      {children}
      {process.env.NODE_ENV === "development" && <ReactQueryDevtools initialIsOpen={false} />}
    </QueryClientProvider>
  );
}
```

Wire it up in `src/app/layout.tsx`:
```tsx
import { Providers } from "@/components/providers";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

**React Query hook pattern** — one hook per query, lives in `feature/hooks/`:

```tsx
// src/features/posts/hooks/use-posts.ts
"use client";

import { useQuery } from "@tanstack/react-query";
import { fetchPosts } from "@/features/posts/api";

export function usePosts() {
  return useQuery({
    queryKey: ["posts"],
    queryFn: fetchPosts,
  });
}
```

**Mutation hook pattern** — invalidate queries on success:

```tsx
// src/features/posts/hooks/use-create-post.ts
"use client";

import { useMutation, useQueryClient } from "@tanstack/react-query";
import { createPost } from "@/features/posts/api";
import type { CreatePostInput } from "@/features/posts/schema";

export function useCreatePost() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (input: CreatePostInput) => createPost(input),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["posts"] });
    },
  });
}
```

**Zustand store pattern** — one store per concern:

```tsx
// src/lib/store/use-sidebar.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

type SidebarState = {
  open: boolean;
  toggle: () => void;
  setOpen: (open: boolean) => void;
};

export const useSidebar = create<SidebarState>()(
  persist(
    (set) => ({
      open: true,
      toggle: () => set((s) => ({ open: !s.open })),
      setOpen: (open) => set({ open }),
    }),
    { name: "sidebar-state" }
  )
);
```

**Rules:**
- **One `QueryClient` per app**, created with `useState` inside a client provider (prevents recreating on re-renders)
- **`staleTime: 60 * 1000`** default — kills the "every component re-render triggers a refetch" footgun
- **Query keys are arrays, not strings.** `["posts", { filter: "published" }]` not `"posts-published"` — easier to partial-invalidate
- **Mutation `onSuccess` invalidates the related query** — don't manually patch the cache unless you've measured a need
- **One Zustand store per concern** — `useSidebar`, `useTheme`, `useFilters`. Don't make a god store
- **`persist` only when state should survive page reloads** — most client state shouldn't

---

### 10. API client pattern

If the frontend talks to an external API (your own backend on a different service, a third-party REST API), use a typed fetch wrapper. Skip this section if the app is purely static.

**`src/lib/api/client.ts`** — base fetch wrapper with Zod response validation:

```typescript
import { z } from "zod";

const API_BASE = process.env.NEXT_PUBLIC_API_URL ?? "";

export class ApiError extends Error {
  constructor(public status: number, public code: string, message: string) {
    super(message);
    this.name = "ApiError";
  }
}

type RequestOptions<TResponse> = {
  method?: "GET" | "POST" | "PATCH" | "PUT" | "DELETE";
  body?: unknown;
  responseSchema?: z.ZodSchema<TResponse>;
  headers?: Record<string, string>;
};

export async function apiFetch<TResponse>(
  path: string,
  options: RequestOptions<TResponse> = {}
): Promise<TResponse> {
  const { method = "GET", body, responseSchema, headers = {} } = options;

  const res = await fetch(`${API_BASE}${path}`, {
    method,
    headers: {
      "content-type": "application/json",
      ...headers,
    },
    body: body ? JSON.stringify(body) : undefined,
  });

  if (!res.ok) {
    const error = await res.json().catch(() => ({}));
    throw new ApiError(
      res.status,
      error?.error?.code ?? "unknown_error",
      error?.error?.message ?? `Request failed: ${res.status}`
    );
  }

  if (res.status === 204) return undefined as TResponse;

  const data = await res.json();

  // If a response schema was passed, validate. Otherwise trust the caller's
  // type assertion. Validation is mandatory for external APIs you don't
  // control — they can change shape under you.
  if (responseSchema) {
    const parsed = responseSchema.safeParse(data);
    if (!parsed.success) {
      throw new ApiError(500, "invalid_response", "API returned unexpected data shape");
    }
    return parsed.data;
  }

  return data as TResponse;
}
```

**Per-feature API module:**

```typescript
// src/features/posts/api.ts
import { apiFetch } from "@/lib/api/client";
import { PostSchema, PostsResponseSchema, type CreatePostInput, type Post } from "./schema";

export function fetchPosts(): Promise<Post[]> {
  return apiFetch("/posts", { responseSchema: PostsResponseSchema });
}

export function fetchPost(id: string): Promise<Post> {
  return apiFetch(`/posts/${id}`, { responseSchema: PostSchema });
}

export function createPost(input: CreatePostInput): Promise<Post> {
  return apiFetch("/posts", {
    method: "POST",
    body: input,
    responseSchema: PostSchema,
  });
}
```

**Rules:**
- **Always validate external API responses with Zod** — they can change shape without telling you
- **Throw `ApiError`** on non-2xx — React Query catches it and exposes `error.status`, `error.code` to your UI
- **Auth tokens go through a separate `useAuthToken()` hook** that injects `Authorization: Bearer ...` — never hardcode tokens in the fetch wrapper
- **`NEXT_PUBLIC_API_URL`** is required if calling an external API. Document it in `.env.example`

---

### 11. Feature folder pattern

```
src/features/<name>/
  components/         ← server + client components for this feature
  hooks/              ← React Query hooks, custom UI hooks
  schema.ts           ← Zod schemas (form input + API response shapes)
  api.ts              ← Typed fetch wrappers (if this feature calls an external API)
  types.ts            ← Shared types (if not derivable from Zod schemas)
  index.ts            ← Public barrel
```

**Rules:**
- **Cross-feature imports go through `index.ts`** — never deep-imports past another feature's barrel
- **One Zod schema, two consumers** — form `zodResolver` and `apiFetch({ responseSchema })`
- **Server components live in `components/`** and can import from `api.ts` directly (fetch at render time). They serve as the initial load; React Query takes over for refetches.
- **Client components import hooks from `hooks/`**, not API functions directly. Mutations and queries go through the hook layer for cache coherence.

---

### 12. App structure

```
src/
  app/
    (marketing)/         ← public landing, pricing, about
      layout.tsx
      page.tsx
    (app)/               ← authenticated UI (if you have client-side auth)
      layout.tsx
      dashboard/
        page.tsx
    api/
      health/route.ts
      log/client-error/route.ts
    error.tsx
    global-error.tsx
    not-found.tsx
    sitemap.ts
    robots.ts
    layout.tsx           ← root layout with Providers
    page.tsx             ← home
  features/
    <feature-name>/
  lib/
    api/
      client.ts
    log.ts
    report-client-error.ts
    utils.ts
    store/
      use-sidebar.ts
  components/
    providers.tsx
    ui/                  ← shadcn components
  instrumentation.ts
```

**`src/app/api/health/route.ts`** — for deploy health checks:
```typescript
import { NextResponse } from "next/server";

export const runtime = "nodejs";

export function GET() {
  return NextResponse.json({ ok: true });
}
```

---

### 13. Vitest

Create `vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import path from "path";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./src/test/setup.ts"],
    env: {
      NEXT_PUBLIC_APP_URL: "https://app.test",
      NEXT_PUBLIC_API_URL: "https://api.test",
    },
  },
  resolve: {
    alias: { "@": path.resolve(__dirname, "./src") },
  },
});
```

Create `src/test/setup.ts`:
```typescript
import "@testing-library/jest-dom/vitest";
import { afterEach } from "vitest";
import { cleanup } from "@testing-library/react";

afterEach(() => {
  cleanup();
});
```

Add to `package.json`:
```json
{ "test": "vitest run" }
```

**Test patterns:**
- **Hooks** — wrap in `QueryClientProvider` and use `renderHook` from `@testing-library/react`
- **Components** — `render` from `@testing-library/react` + `screen.getByRole(...)`
- **Schemas** — direct `Schema.safeParse(...)` assertions, no rendering needed
- **API client** — mock `fetch` globally with `vi.fn()`

---

### 14. Environment variables

Create `.env.example`:
```bash
# App URL (used for canonical metadataBase, sitemap, robots)
NEXT_PUBLIC_APP_URL=http://localhost:3000

# External API (skip if purely static)
NEXT_PUBLIC_API_URL=https://api.example.com

# Logging
LOG_LEVEL=debug
```

Create `.env.local` from `.env.example`.

Add to `.gitignore`:
```
.env.local
.env.production
.env*.local
```

---

### 15. TypeScript config

`tsconfig.json` — ensure these are set:
```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "paths": { "@/*": ["./src/*"] }
  }
}
```

---

### 16. Styling conventions

**Component file naming — always kebab-case:**
```
✅ home-navbar.tsx     ✅ user-avatar.tsx
❌ HomeNavbar.tsx      ❌ UserAvatar.tsx
```

The exported component name is still PascalCase:
```tsx
// home-navbar.tsx
export function HomeNavbar() { ... }
```

---

**No hardcoded color values — ever:**
```tsx
// ❌
<div className="bg-[#1a1a1a] text-white border-gray-200">
<p className="text-red-500">Error</p>

// ✅ Semantic tokens from globals.css
<div className="bg-background text-foreground border-border">
<p className="text-destructive">Error</p>
```

Token set: `background`, `foreground`, `card`, `primary`, `secondary`, `muted`, `accent`, `destructive`, `border`, `input`, `ring`.

If reaching for `text-gray-500` → use `text-muted-foreground`. `text-red-500` → `text-destructive`. `bg-white` → `bg-background`.

---

**No hardcoded `px` values — use Tailwind scale:**
```tsx
// ❌
<div className="p-[14px] mt-[32px] w-[320px] text-[13px]">

// ✅
<div className="p-3.5 mt-8 w-80 text-sm">
```

**Width and height — prefer relative:**
```tsx
// ❌ Fixed widths break on small screens
<div className="w-[1200px]">

// ✅
<div className="w-full max-w-5xl">          // constrained but fluid
<div className="h-screen">                  // viewport-relative
<div className="w-full md:w-1/2 lg:w-1/3">  // responsive columns
```

**Responsive design — mobile-first:**
```tsx
<div className="flex flex-col gap-4 md:flex-row md:gap-6">
<div className="text-base md:text-lg lg:text-xl">
```

---

**`page.tsx` files never use `"use client"`:**
```tsx
// ❌ Kills SSR — crawler sees empty shell instead of content
"use client";
export default function ProductPage() { /* ... */ }

// ✅ Page stays server-rendered. Interactivity goes in a child marked "use client"
import { AddToCartButton } from "./_components/add-to-cart-button";

export default async function ProductPage() {
  const product = await fetchProduct(); // server-side data fetch
  return (
    <>
      <h1>{product.name}</h1>
      <AddToCartButton productId={product.id} />
    </>
  );
}
```

**Rule of thumb:** if the file is `page.tsx`, `layout.tsx`, or any route entry, it stays a server component. Push interactivity into child client components.

---

### 17. Logger conventions

Pino is the **only** logger for server-side code (server components, route handlers, instrumentation). `console.log` is banned server-side.

**Always: context object first, message string second**
```typescript
log.info({ userId, postId: row.id }, "posts.created");
log.error({ url, err }, "external_api.fetch_failed");

// ❌ String interpolation kills structured search
log.info(`Created post ${row.id}`);
```

**Event name format: `feature.verb`**
```
"posts.created"     "external_api.fetch_failed"     "client.error"
```

**Log levels:**

| Level | When |
|---|---|
| `log.fatal` | Process is going down |
| `log.error` | Server-side failure the user/system shouldn't ignore |
| `log.warn` | Recoverable degradation |
| `log.info` | Meaningful state change (rare in frontend-only) |
| `log.debug` | Dev-only detail |

**Client-side errors** — use `reportClientError()`, never `console.error`:
```tsx
import { toast } from "sonner";
import { reportClientError } from "@/lib/report-client-error";

try {
  await mutate();
} catch (err) {
  reportClientError(err, { feature: "posts" });
  toast.error("Something went wrong. Please try again.");
}
```

---

### 18. STANDARDS.md

Create `STANDARDS.md`:

```markdown
# STANDARDS.md

The engineering rules for this repo. Every code change should satisfy them.

1. **No `any`, no `as` casts on user input** — validate with Zod at every boundary
2. **`safeParse` everywhere, `firstError` for the user message** — never `.parse()`
3. **No `console.log` server-side** — use `log` from `@/lib/log`
4. **No `console.error` client-side** — use `reportClientError()` + `toast.error()`
5. **Validate every external API response with Zod** — they can change shape without warning
6. **Commit messages: `type: description`** — enforced by commit-msg hook
7. **Feature folders: `components → hooks → schema → api → index`** — no exceptions
8. **Cross-feature imports go through `index.ts`** — never deep-import into another feature
9. **`"use server"` only at the top of files, never inline** — extract into a dedicated file
10. **`page.tsx` files never use `"use client"`** — pages stay server components for SSR/SEO. Extract interactivity into child components
11. **Component filenames are kebab-case** — `home-navbar.tsx`, not `HomeNavbar.tsx`
12. **No hardcoded color values** — use semantic tokens only (`bg-background`, `text-muted-foreground`)
13. **No hardcoded `px` values** — use Tailwind spacing scale
14. **Use shadcn components, never duplicate them** — extend via `cva` variants
15. **Every form uses `react-hook-form` + `zodResolver`** — same Zod schema as the API request validation
16. **Server state goes through React Query, not `useState`/`useEffect`** — `useState` fetching causes race conditions
17. **Client state goes through Zustand** — one store per concern, not a god store
18. **Static assets live in `public/`, referenced with absolute paths** — `/logo.png` not `./logo.png`. No secrets, no files >1MB
19. **Use `next/image` for raster images and `next/font` for fonts** — never `<img>` or self-hosted font files
20. **Test schemas, hooks, and key components** — happy path + error path minimum
```

---

### 19. CLAUDE.md

Create `CLAUDE.md` at repo root:

```markdown
# CLAUDE.md

Auto-loaded by Claude Code. Companion files:
- [`STANDARDS.md`](./STANDARDS.md) — engineering rules
- [`REVIEW.md`](./REVIEW.md) — review rubric
- [`AGENTS.md`](./AGENTS.md) — AI review bot entry point

---

## Never run git commands — show them instead

Never execute `git add`, `git commit`, `git push`, or any mutating git command. Paste the exact commands for the user to run instead.

---

## Repo orientation

Frontend-only Next.js 16 app. Bun = package manager, Node.js = runtime.

| Surface | Where |
|---|---|
| Marketing / public pages | `src/app/(marketing)/` |
| Authenticated UI (if applicable) | `src/app/(app)/` |
| Minimal API routes (health, client error reporting) | `src/app/api/` |
| Business logic / data fetching | `src/features/<name>/` |
| Shared utilities | `src/lib/` |

---

## Feature folder reference

Every feature follows:
```
src/features/<name>/
  components/
  hooks/          ← React Query hooks
  schema.ts       ← Zod schemas
  api.ts          ← Typed fetch wrappers (if hitting an external API)
  index.ts
```

Cross-feature imports go through `index.ts`. Never deep-import.

---

## Gotchas

**`page.tsx` files never use `"use client"`.** Adding it kills server rendering — the crawler sees an empty shell. Extract interactivity into a child component (e.g. `_components/add-to-cart-button.tsx`) and import it. Same rule for `layout.tsx`.

**Server state through React Query, not `useState`.** `useState` + `useEffect` fetching has race conditions, no caching, and re-fetches on every mount. Always React Query.

**Validate external API responses with Zod.** External APIs change shape without notice. The `apiFetch({ responseSchema })` pattern catches drift at the boundary instead of letting it cause cryptic UI errors three components deep.

---

## Commands

| Command | What |
|---|---|
| `bun install` | Install deps |
| `bun run dev` | Start dev server on :3000 |
| `bun run build` | Production build |
| `bunx vitest run` | Run tests |
| `bunx tsc --noEmit` | Typecheck |
| `bunx next lint` | ESLint |
| `bun audit --audit-level=high` | Security audit |
```

---

### 20. AGENTS.md

Create `AGENTS.md`:

```markdown
# AGENTS.md

Entry point for AI code-review agents (Codex, Gemini Code Assist, Claude Code, Devin).

Read these in order before reviewing:
1. [`CLAUDE.md`](./CLAUDE.md) — repo orientation
2. [`STANDARDS.md`](./STANDARDS.md) — engineering rules
3. [`REVIEW.md`](./REVIEW.md) — full review rubric

---

## Review guidelines

### 🔴 Block-merge (always raise)

- **`"use client"` on a `page.tsx` or `layout.tsx`** — kills SSR/SEO
- **External API response not validated with Zod** — drift becomes a runtime bug
- **Form not using `react-hook-form` + `zodResolver`** — `useState` form state is banned
- **Data fetching with `useState` + `useEffect`** instead of React Query
- **Hardcoded secrets in code or committed env files** — even `NEXT_PUBLIC_*` for sensitive values is wrong
- **Module-load throws for runtime env vars** — breaks `next build`'s page-data collection
- **`<img>` instead of `next/image` for raster images** — kills LCP scores

### 🟡 Discuss / suggest

- Hardcoded color values (`text-red-500`, `bg-[#1a1a1a]`) — use semantic tokens
- Hardcoded `px` values (`p-[14px]`, `w-[320px]`) — use Tailwind scale
- Component filename in PascalCase — should be kebab-case
- `console.log`/`console.error` in client code — use `reportClientError` + `toast.error`
- New Zustand store added without checking if state actually needs to persist
- React Query `staleTime` not set on a query (defaults to 0 — refetches aggressively)
- Missing `defaultValues` on `useForm` — triggers "uncontrolled to controlled" warnings

### ⚪ Skip / don't comment

- Style nits auto-fixed by Prettier
- Missing tests for trivial helpers
- Outdated transitive dependencies (handled separately)

---

## Files to skip entirely

| Path | Reason |
|---|---|
| `bun.lock`, `package-lock.json` | Lockfiles |
| `**/*.test.ts`, `**/*.test.tsx` | Tests reviewed by humans |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Context, not code |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |
| `**/public/images/**` | Static assets |

---

## Confidence calibration

- Read the file, not just the diff
- Don't suggest renames or refactors that aren't behavior changes
- Don't recommend a component that doesn't exist
- Match existing patterns
- If unsure, don't flag unless it's in 🔴 or 🟡
```

---

### 21. REVIEW.md

Create `REVIEW.md`:

```markdown
# REVIEW.md

Code-review rubric. Applies to humans and bots.

Read [`CLAUDE.md`](./CLAUDE.md) and [`STANDARDS.md`](./STANDARDS.md) first.

---

## Severity rubric

### 🔴 Block-merge

- **`"use client"` on a `page.tsx` / `layout.tsx`** — kills SSR/SEO
- **External API response without Zod validation** — drift causes runtime bugs
- **Form without `react-hook-form` + `zodResolver`** — `useState` form state is banned
- **Server state fetched with `useState` + `useEffect`** instead of React Query
- **Module-load throws for env vars** — breaks `next build`
- **Hardcoded secrets** in code or env files committed to git
- **`<img>` instead of `next/image`** for raster images

### 🟡 Discuss

- Hardcoded color / px values — use semantic tokens / Tailwind scale
- Component filename in PascalCase — should be kebab-case
- `console.*` in client code — use `reportClientError` + `toast.error`
- React Query `staleTime` not set — defaults to 0 and refetches aggressively
- Form `useForm` without `defaultValues` — triggers controlled-input warnings
- Cross-feature deep imports — go through `index.ts`
- Native `confirm()` / `alert()` — use shadcn `Dialog`
- Duplicate shadcn components — extend via `cva` variants

### ⚪ Skip

- Auto-fixable style nits
- Theoretical edge cases without a concrete path
- Missing tests for trivial helpers
- Outdated transitive dependencies (handled separately)

### Files to skip entirely

| Path | Reason |
|---|---|
| `bun.lock`, `package-lock.json` | Lockfiles |
| `**/*.test.ts(x)` | Reviewed by humans |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Context |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |
| `**/public/**` | Static assets |

---

## Confidence calibration

- Read the file, not just the diff
- Don't suggest refactors that aren't behavior changes
- Don't recommend components that don't exist — check `src/components/ui/`
- Match existing patterns
- If unsure, don't flag unless 🔴 or 🟡
```

---

### 22. PR template

Create `.github/pull_request_template.md`:

```markdown
## Summary

<!-- 1-3 bullets: what changed and why -->

-

## Checklist

- [ ] Followed [STANDARDS.md](../STANDARDS.md)
- [ ] Tests pass locally (`bunx vitest run`)
- [ ] Typecheck + lint pass (`bunx tsc --noEmit`, `bunx next lint`)

## Test plan

<!-- How to verify this works -->

-
```

---

### 23. `next.config.ts`

Create `next.config.ts`:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // Required for Docker / Cloud Run deploys — builds a self-contained
  // output directory instead of relying on node_modules at runtime.
  output: "standalone",

  images: {
    remotePatterns: [
      // Add every external image domain here. Never use dangerouslyAllowSVG
      // or wildcard hostnames.
      // { protocol: "https", hostname: "images.example.com" },
    ],
  },

  // Security headers — applied to every response
  async headers() {
    return [
      {
        source: "/(.*)",
        headers: [
          { key: "X-Frame-Options", value: "DENY" },
          { key: "X-Content-Type-Options", value: "nosniff" },
          { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
          { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
        ],
      },
    ];
  },
};

export default nextConfig;
```

---

### 24. `instrumentation.ts`

Create `src/instrumentation.ts`:

```typescript
export async function register(): Promise<void> {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;

  const { log } = await import("./lib/log");

  process.on("uncaughtException", (err) => {
    log.fatal({ err }, "process.uncaught_exception");
    process.exit(1);
  });

  process.on("unhandledRejection", (reason) => {
    log.fatal({ reason }, "process.unhandled_rejection");
  });

  process.on("SIGTERM", () => {
    log.info("process.sigterm — shutting down");
    process.exit(0);
  });

  const REQUIRED = ["NEXT_PUBLIC_APP_URL"];

  for (const key of REQUIRED) {
    if (!process.env[key]) {
      log.error({ key }, `instrumentation.missing_env — ${key} is not set`);
    }
  }
}

export async function onRequestError(
  err: unknown,
  request: { path: string; method: string; headers: { [key: string]: string | undefined } },
  context: {
    routerKind: "Pages Router" | "App Router";
    routePath: string;
    routeType: "render" | "route" | "action" | "middleware";
  }
): Promise<void> {
  const { log } = await import("./lib/log");
  const error = err instanceof Error ? err : new Error(String(err));
  const digest = (error as Error & { digest?: string }).digest;

  log.error(
    {
      err: error,
      digest,
      request: { path: request.path, method: request.method },
      route: { kind: context.routerKind, path: context.routePath, type: context.routeType },
    },
    "request.error"
  );
}
```

---

### 25. Error + loading pages

**`src/app/error.tsx`:**
```tsx
"use client";

import { useEffect } from "react";
import { Button } from "@/components/ui/button";
import { reportClientError } from "@/lib/report-client-error";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    reportClientError(error, { digest: error.digest });
  }, [error]);

  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-lg font-semibold">Something went wrong</h2>
      {error.digest && (
        <p className="text-sm text-muted-foreground">Reference: {error.digest}</p>
      )}
      <Button onClick={reset}>Try again</Button>
    </div>
  );
}
```

**`src/app/global-error.tsx`:**
```tsx
"use client";

import { Button } from "@/components/ui/button";

export default function GlobalError({ reset }: { reset: () => void }) {
  return (
    <html>
      <body className="flex min-h-screen flex-col items-center justify-center gap-4">
        <h2 className="text-lg font-semibold">Something went wrong</h2>
        <Button onClick={reset}>Try again</Button>
      </body>
    </html>
  );
}
```

**`src/app/not-found.tsx`:**
```tsx
import Link from "next/link";
import { Button } from "@/components/ui/button";

export default function NotFound() {
  return (
    <div className="flex min-h-screen flex-col items-center justify-center gap-4">
      <h2 className="text-lg font-semibold">Page not found</h2>
      <Button asChild variant="outline">
        <Link href="/">Go home</Link>
      </Button>
    </div>
  );
}
```

**`src/app/loading.tsx`:**
```tsx
export default function Loading() {
  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="h-6 w-6 animate-spin rounded-full border-2 border-muted-foreground border-t-transparent" />
    </div>
  );
}
```

---

### 26. SEO + assets

#### Root-layout metadata

**`src/app/layout.tsx`:**
```tsx
import type { Metadata } from "next";
import { Providers } from "@/components/providers";
import "./globals.css";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export const metadata: Metadata = {
  metadataBase: new URL(APP_URL),
  title: {
    default: "{{APP_NAME}}",
    template: "%s | {{APP_NAME}}",
  },
  description: "Short, accurate description of what {{APP_NAME}} does.",
  openGraph: {
    title: "{{APP_NAME}}",
    description: "Short, accurate description of what {{APP_NAME}} does.",
    type: "website",
    url: APP_URL,
    siteName: "{{APP_NAME}}",
    images: [{ url: "/opengraph-image", width: 1200, height: 630 }],
  },
  twitter: {
    card: "summary_large_image",
    title: "{{APP_NAME}}",
    description: "Short, accurate description of what {{APP_NAME}} does.",
    images: ["/opengraph-image"],
  },
  robots: {
    index: true,
    follow: true,
    googleBot: { index: true, follow: true, "max-image-preview": "large" },
  },
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

#### Per-page metadata

```tsx
// src/app/blog/[slug]/page.tsx
import type { Metadata } from "next";
import { fetchPost } from "@/features/posts/api";

export async function generateMetadata({
  params,
}: {
  params: Promise<{ slug: string }>;
}): Promise<Metadata> {
  const { slug } = await params;
  const post = await fetchPost(slug);
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage ?? "/opengraph-image"],
    },
  };
}
```

#### Static generation for dynamic routes

For routes you can enumerate at build time (blog posts, marketing pages), use `generateStaticParams`:

```tsx
// src/app/blog/[slug]/page.tsx
export async function generateStaticParams() {
  const posts = await fetchAllPostSlugs();
  return posts.map((slug) => ({ slug }));
}
```

This produces a static HTML file per route at build time — fastest possible TTFB and SEO ranking.

#### `sitemap.ts`

**`src/app/sitemap.ts`:**
```typescript
import type { MetadataRoute } from "next";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const staticRoutes: MetadataRoute.Sitemap = [
    { url: APP_URL, lastModified: new Date(), changeFrequency: "daily", priority: 1.0 },
    { url: `${APP_URL}/about`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.5 },
  ];

  // For dynamic content, fetch and map:
  // const posts = await fetchAllPostSlugs();
  // const blogRoutes = posts.map((slug) => ({
  //   url: `${APP_URL}/blog/${slug}`,
  //   lastModified: new Date(),
  //   changeFrequency: "weekly" as const,
  //   priority: 0.7,
  // }));

  return staticRoutes;
}
```

#### `robots.ts`

```typescript
import type { MetadataRoute } from "next";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: "*",
        allow: "/",
        disallow: ["/api/", "/(app)/"],
      },
    ],
    sitemap: `${APP_URL}/sitemap.xml`,
    host: APP_URL,
  };
}
```

#### Favicon + OG image conventions

Next.js auto-serves these from `src/app/`:

| Filename | Purpose | Size |
|---|---|---|
| `icon.png` | Browser tab favicon | 32×32 or 512×512 |
| `apple-icon.png` | iOS home-screen icon | 180×180 |
| `opengraph-image.png` | Default OG (Facebook, LinkedIn, Slack) | 1200×630 |
| `twitter-image.png` | Twitter card | 1200×600 |

**Default OG = upscaled favicon** on a brand-colored background. Use [realfavicongenerator.net](https://realfavicongenerator.net) or any image editor.

#### Public folder — asset management

```tsx
// ✅ Absolute paths from /public
<img src="/logo.svg" alt="Logo" />
<Image src="/images/hero.jpg" alt="Hero" width={1200} height={600} />

// ❌ Relative paths break under different routes
<img src="./logo.svg" />
```

**Recommended structure:**
```
public/
  images/          ← photos, illustrations
  icons/           ← custom SVG icons
  videos/          ← MP4s, WebM
  documents/       ← PDFs user can download
```

**Rules:**
- **No secrets in `public/`** — every file is served unconditionally
- **No files >1MB in `public/`** — bloats the deploy and slows first paint. Use a CDN for big assets
- **Prefer `next/font` over self-hosted font files** — automatic preloading, no FOUT
- **Use `next/image` for raster images** — automatic responsive sizing, lazy loading, WebP/AVIF
- **SVG icons** — inline React components for ones you reuse; `public/icons/` for one-off illustrations

---

### 27. CI workflow

Create `.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: ["**"]

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}

env:
  NEXT_TELEMETRY_DISABLED: 1

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx next lint

  format:
    name: Format Check
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx prettier --check "**/*.{ts,tsx,js,jsx,json,md,css}"

  typecheck:
    name: Typecheck
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx tsc --noEmit

  audit:
    name: Security Audit
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bun audit --audit-level=high

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx vitest run

  build:
    name: Build
    runs-on: ubuntu-latest
    timeout-minutes: 20
    needs: [lint, format, typecheck, audit, test]
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - name: Cache deps
        uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx next build
        env:
          NEXT_PUBLIC_APP_URL: https://app.test
```

---

### 28. Vercel deployment

**One-time setup:**
1. `vercel link` from the repo root
2. Grab the project ID and org ID from `.vercel/project.json`
3. Create a Vercel API token at [vercel.com/account/tokens](https://vercel.com/account/tokens)
4. Add three repo secrets in GitHub → Settings → Secrets → Actions:
   - `VERCEL_TOKEN`
   - `VERCEL_ORG_ID`
   - `VERCEL_PROJECT_ID`

**`vercel.json`** — disable Vercel's auto-git deploy so the workflow is the only deploy path:
```json
{
  "git": {
    "deploymentEnabled": {
      "main": false
    }
  }
}
```

**`.github/workflows/deploy.yml`:**
```yaml
name: Vercel Production Deployment

env:
  VERCEL_PROJECT_ID: ${{ secrets.VERCEL_PROJECT_ID }}
  VERCEL_ORG_ID: ${{ secrets.VERCEL_ORG_ID }}

on:
  push:
    branches:
      - main

permissions:
  contents: read
  deployments: write

jobs:
  deploy-production:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Verify secrets
        run: |
          test -n "${{ secrets.VERCEL_TOKEN }}" || (echo "Missing VERCEL_TOKEN" && exit 1)
          test -n "${{ env.VERCEL_ORG_ID }}" || (echo "Missing VERCEL_ORG_ID" && exit 1)
          test -n "${{ env.VERCEL_PROJECT_ID }}" || (echo "Missing VERCEL_PROJECT_ID" && exit 1)

      - name: Install Vercel CLI
        run: npm i -g vercel@latest

      - name: Vercel pull (production)
        run: vercel pull --yes --environment=production --token ${{ secrets.VERCEL_TOKEN }}

      - name: Vercel build (production)
        run: vercel build --prod --token ${{ secrets.VERCEL_TOKEN }}

      - name: Remove Git metadata
        run: |
          rm -rf .git
          unset GITHUB_ACTOR GITHUB_SHA GITHUB_REF GITHUB_HEAD_REF GITHUB_REPOSITORY

      - name: Vercel deploy (production)
        run: vercel deploy --prebuilt --prod --yes --token ${{ secrets.VERCEL_TOKEN }}
```

Set all env vars from `.env.example` in Vercel → Project Settings → Environment Variables → Production scope. The workflow's `vercel pull` step downloads them into the build.

---

### 29. Dockerfile + .dockerignore

For deploys outside Vercel (Cloud Run, Fly, Railway, self-hosted).

**`Dockerfile`:**
```dockerfile
# syntax=docker/dockerfile:1.6

FROM oven/bun:1 AS deps
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

FROM oven/bun:1 AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG NEXT_PUBLIC_APP_URL
ARG NEXT_PUBLIC_API_URL
ENV NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ENV NEXT_TELEMETRY_DISABLED=1

RUN bun run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

RUN addgroup -S nodejs && adduser -S nextjs -G nodejs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

**`.dockerignore`:**
```
node_modules
.next
.git
.github
.husky
.vscode
.idea
.DS_Store
*.log
*.lock
*.tsbuildinfo
.env
.env.local
.env*.local
coverage
**/dist
**/build
README.md
*.md
Thumbs.db
.vercel
```

---

### 30. .gitignore additions

```
# Env
.env.local
.env.production
.env*.local

# Build output
.next/
out/
build/

# Vercel
.vercel
```

---

### 31. Final checks

Run in order:
```bash
bun install
bunx tsc --noEmit
bunx next lint
bunx vitest run
bunx next build
```

All should pass before the first commit.

First commit:
```bash
git add .
git commit -m "chore: initial project setup"
```

---

## What this scaffold gives you

| Piece | What it does |
|---|---|
| Bun + Next.js 16 | Fast installs, Node.js runtime |
| Husky hooks | Format, lint, types, tests, audit on every push |
| Conventional commits | Enforced by commit-msg hook |
| shadcn/ui + tweakcn theme | Full token set, no hardcoded colors |
| Zod conventions | `safeParse` everywhere, user-facing messages |
| Forms | `react-hook-form` + `zodResolver` — same schema as API validation |
| React Query | Server state with cache, no `useState` fetching |
| Zustand | Client-only state, one store per concern |
| API client | Typed fetch wrapper, Zod response validation |
| Feature folder pattern | `components → hooks → schema → api → index` |
| Vitest + Testing Library | Component, hook, schema tests |
| Pino logger | Structured JSON for SSR errors |
| Client error reporting | `/api/log/client-error` + `reportClientError()` |
| Styling conventions | Kebab-case files, no hardcoded px/colors, mobile-first |
| `next.config.ts` | `standalone` output, security headers |
| `instrumentation.ts` | Env validation + `onRequestError` |
| Error/loading pages | `error.tsx`, `global-error.tsx`, `not-found.tsx`, `loading.tsx` |
| SEO | `sitemap.ts`, `robots.ts`, OG image conventions, per-page metadata |
| Static generation | `generateStaticParams` for dynamic routes (fastest TTFB) |
| Governance docs | `CLAUDE.md`, `STANDARDS.md`, `AGENTS.md`, `REVIEW.md` |
| Vercel deploy | GitHub Actions workflow + `vercel.json` disabling auto-deploy |
| Dockerfile | Multi-stage Bun→Node build for non-Vercel hosts |
| CI workflow | Parallel lint/format/typecheck/audit/test jobs |

## PROMPT END
