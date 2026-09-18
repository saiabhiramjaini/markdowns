# Next.js Frontend-Only Scaffold Prompt

Copy-paste this entire prompt to scaffold a production-ready **frontend-only** Next.js app from scratch.
Replace `{{APP_NAME}}` with your app name (kebab-case, e.g. `my-app`).

**Use this when:** the app has no first-party database, no NextAuth, no server-side mutations. It may call an **external** API you do not control, or be purely static (marketing, docs, landing).

**Use the server-actions prompt instead when:** you need Postgres, NextAuth, and mutations in this Next app.

**Use the REST `/api/v1` prompt instead when:** this Next app *is* the API for mobile / third-party clients.

---

## PROMPT START

Scaffold a production-ready frontend-only Next.js app called `{{APP_NAME}}`. Follow every instruction exactly — don't add extras, don't skip steps.

Non-interactive CLIs only: `bunx shadcn@latest init -d`. Do not open tweakcn / Vercel in a browser. Ship a default `globals.css` token set (shadcn Slate + CSS variables).

**Write `CLAUDE.md`, `AGENTS.md`, `STANDARDS.md`, and `REVIEW.md` in full from the templates in this prompt.** Do not stub. They must match §0. Do **not** add Sentry, PostHog, Redis, NextAuth, Drizzle, or a response cache.

---

### 0. Exception handling, errors, logging, observability (read first)

Three layers (there is no audit table and no first-party API). Do not mix them.

```
thrown Error in SSR / route     →   onRequestError + generic UI
external API fail / Zod drift   →   user-safe toast + log.error
log.info / warn / error         →   stdout JSON. Operators only.
reportClientError(...)          →   POST /api/log/client-error → same Pino line
```

**Exceptions**

- Server components and the few routes (`/api/health`, `/api/log/client-error`, optional `/api/contact`) must not leak `err.message` or stacks to the browser.
- `error.tsx` shows a generic message + `digest`. `reportClientError` sends the stack to the server log.
- Process: `uncaughtException` → `log.fatal` in Node-only `process-handlers.ts` (not inside `instrumentation.ts`). `unhandledRejection` → `log.error`.

**Error handling (what the user sees)**

- Forms: Zod via `zodResolver` + `toast.error` / `form.setError`. Never a raw fetch error string.
- External API: `apiFetch` `safeParse`s the response. Drift → `ApiError` / toast, not a crash three components down.
- No `"use server"` mutations. If you need those, you picked the wrong prompt.

**Logging**

Pino only on the server. `console.*` is an ESLint error in app code. Signature: `log.<level>({ ...context }, "feature.verb")`.

| Level | When |
|---|---|
| `fatal` | Process is going down |
| `error` | SSR / route / external fetch actually failed; client error reported |
| `warn` | Recoverable |
| `info` | Rare on a frontend-only app (contact submitted) |
| `debug` | Off in prod unless `LOG_LEVEL=debug` |

IP on `client.error_reported` / `request.error` only — not on every line. React Query is the client cache (`staleTime` + `invalidateQueries`). Do not add Redis.

**Observability**

- Dual fields: `level` + GCP `severity`. `service` from `DD_SERVICE`.
- `APP_ENV=local|dev|prod` is a **build ARG**. Browser source maps on local/dev, **off prod**.
- `/api/health` returns `{ ok: true }` (no DB). Probe this, not `/`.
- Do not add Sentry or PostHog until a DSN/key exists.

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

**Step 1 — Init shadcn (non-interactive):**
```bash
bunx shadcn@latest init -d
```

Defaults (Default style, Neutral/Slate, CSS variables) are fine. Do not wait for a TTY.

**Step 2 — Add the base component set:**
```bash
bunx shadcn@latest add button input label textarea dialog dropdown-menu badge toast card separator skeleton tabs select checkbox radio-group switch tooltip popover sheet command avatar form
```

**Step 3 — Theme:** keep the shadcn `globals.css` token set. Optionally replace later from tweakcn.com — not part of codegen.

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
      return { service: process.env.DD_SERVICE ?? "{{APP_NAME}}" };
    },
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: [
      "password", "*.password", "token", "*.token",
      "apiKey", "*.apiKey", "authorization", "*.authorization",
      "headers.authorization", "headers.cookie",
    ],
    censor: "[REDACTED]",
    remove: false,
  },
};

const devTransport: LoggerOptions["transport"] =
  !isProd && !isTest
    ? { target: "pino-pretty", options: { colorize: true, translateTime: "SYS:HH:MM:ss.l", ignore: "pid,hostname,service,severity" } }
    : undefined;

function buildDestination(): pino.DestinationStream | undefined {
  const appLogFile = process.env.APP_LOG_FILE;
  if (!appLogFile || devTransport) return undefined;
  try {
    const fileStream = pino.destination({ dest: appLogFile, mkdir: true, sync: false });
    fileStream.on("error", () => {});
    return pino.multistream([
      { stream: pino.destination({ dest: 1, sync: false }) },
      { stream: fileStream },
    ]);
  } catch (err) {
    process.stderr.write(
      `[log] APP_LOG_FILE unusable, stdout only: ${err instanceof Error ? err.message : String(err)}\n`
    );
    return undefined;
  }
}

const destination = buildDestination();

export const log: Logger = devTransport
  ? pino({ ...baseOptions, transport: devTransport })
  : destination
    ? pino(baseOptions, destination)
    : pino(baseOptions);
```

**`src/lib/app-environment.ts`:**
```typescript
export const APP_ENVIRONMENTS = ["local", "dev", "prod"] as const;
export type AppEnvironment = (typeof APP_ENVIRONMENTS)[number];
export type EnvironmentSettings = { enableBrowserSourceMaps: boolean };
export const ENVIRONMENT_SETTINGS: Record<AppEnvironment, EnvironmentSettings> = {
  local: { enableBrowserSourceMaps: true },
  dev: { enableBrowserSourceMaps: true },
  prod: { enableBrowserSourceMaps: false },
};
export function appEnvironment(value: string | undefined): AppEnvironment {
  return APP_ENVIRONMENTS.includes(value as AppEnvironment) ? (value as AppEnvironment) : "local";
}
```

**`src/lib/get-ip.ts`:**
```typescript
import type { NextRequest } from "next/server";
export function getIp(req: NextRequest): string {
  return (
    req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ??
    req.headers.get("x-real-ip") ??
    "unknown"
  );
}
```

**`src/lib/process-handlers.ts`** — Node-only. Not in `instrumentation.ts`.
```typescript
import { log } from "./log";
export function attachProcessHandlers(): void {
  if ((globalThis as { __processHandlers?: boolean }).__processHandlers) return;
  (globalThis as { __processHandlers?: boolean }).__processHandlers = true;
  process.on("uncaughtException", (err) => {
    log.fatal({ err }, "process.uncaught_exception");
  });
  process.on("unhandledRejection", (reason) => {
    log.error(
      { err: reason instanceof Error ? reason : new Error(String(reason)) },
      "process.unhandled_rejection"
    );
  });
}
```

**`src/lib/report-client-error.ts`** — for client-component errors:
```typescript
export function reportClientError(err: unknown, source = "unknown"): void {
  if (typeof window === "undefined") return;
  const message = err instanceof Error ? err.message : String(err);
  const stack = err instanceof Error ? err.stack : undefined;
  fetch("/api/log/client-error", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({
      message: message.slice(0, 2_000),
      stack: stack?.slice(0, 8_000),
      source: typeof source === "string" ? source.slice(0, 50) : "unknown",
      url: window.location.href.slice(0, 2_000),
    }),
    keepalive: true,
    credentials: "same-origin",
  }).catch(() => {});
}
```

**`src/app/api/log/client-error/route.ts`** — same-origin, Zod, size cap, in-memory IP cap:
```typescript
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { log } from "@/lib/log";
import { getIp } from "@/lib/get-ip";

export const runtime = "nodejs";

const Schema = z.object({
  message: z.string().min(1).max(2_000),
  stack: z.string().max(8_000).optional(),
  source: z.string().min(1).max(50),
  url: z.string().max(2_000).optional(),
});

const buckets = new Map<string, { count: number; resetAt: number }>();
function allow(ip: string): boolean {
  const now = Date.now();
  const b = buckets.get(ip);
  if (!b || b.resetAt <= now) {
    buckets.set(ip, { count: 1, resetAt: now + 60_000 });
    return true;
  }
  if (b.count >= 10) return false;
  b.count += 1;
  return true;
}

export async function POST(req: NextRequest): Promise<NextResponse> {
  const origin = req.headers.get("origin");
  const app = process.env.NEXT_PUBLIC_APP_URL;
  if (origin && app && origin !== new URL(app).origin) {
    return new NextResponse(null, { status: 204 });
  }
  if (Number(req.headers.get("content-length") ?? 0) > 12_000) {
    return new NextResponse(null, { status: 413 });
  }
  const ip = getIp(req);
  if (!allow(ip)) return new NextResponse(null, { status: 429 });
  const parsed = Schema.safeParse(await req.json().catch(() => null));
  if (!parsed.success) return new NextResponse(null, { status: 400 });
  const err = new Error(parsed.data.message);
  if (parsed.data.stack) err.stack = parsed.data.stack;
  log.error({ err, source: parsed.data.source, client: { url: parsed.data.url, ip } }, "client.error_reported");
  return new NextResponse(null, { status: 204 });
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
    app-environment.ts
    process-handlers.ts
    get-ip.ts
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
APP_ENV=local
LOG_LEVEL=debug
DD_SERVICE={{APP_NAME}}
# APP_LOG_FILE=/shared-volume/logs/app.log

# External API (skip if purely static)
NEXT_PUBLIC_API_URL=https://api.example.com
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

Write `STANDARDS.md` in full. Do not stub.

```markdown
# STANDARDS.md

## Architecture

- Feature folders: `schema → api → hooks → index → components`. Zustand stores live in `src/lib/store/` or the feature.
- Cross-feature imports go through `index.ts`.
- No first-party DB, NextAuth, Redis, or `"use server"` mutations. If you need those, you picked the wrong prompt.
- `page.tsx` / `layout.tsx` never `"use client"`.
- Server state: React Query. Client UI state: Zustand. Never `useState`+`useEffect` fetching.
- External API responses: Zod `safeParse` via `apiFetch({ responseSchema })`.

## Errors and logging

- Pino on the server. `log.<level>({ ...ctx }, "feature.verb")`. No `console.*` in app code.
- Client: `reportClientError()` + `toast.error`. Never swallow.
- `/api/log/client-error` is same-origin, Zod, size-capped, IP-capped. Reconstruct `Error` so Pino's `err` serializer works.
- IP on client-error / `request.error` only.
- `APP_ENV` drives source maps. Dual `level` + `severity`. No Sentry/PostHog without a DSN.

## UI, tests, git

- kebab-case files. Tokens + Tailwind. shadcn only.
- Forms: `react-hook-form` + `zodResolver`.
- `next/image`, `next/font`, assets in `public/`.
- Commit: `type: description`.
```

---

### 19. CLAUDE.md

Write `CLAUDE.md` in full. No nested fences inside the template.

```markdown
# CLAUDE.md

Companion files: [`AGENTS.md`](./AGENTS.md), [`STANDARDS.md`](./STANDARDS.md), [`REVIEW.md`](./REVIEW.md).

Never commit, push, or open a PR unless the user asks. Never `--no-verify`.

## Repo

Frontend-only Next.js 16. Bun = package manager, Node.js = runtime. No Postgres, no NextAuth, no Redis.

| Surface | Where |
|---|---|
| Marketing | `src/app/(marketing)/` |
| App UI | `src/app/(app)/` if client-side auth against an external API |
| Health | `src/app/api/health/` — `{ ok: true }`, no DB |
| Client errors | `src/app/api/log/client-error/` |
| Features | `src/features/<name>/` |
| External API client | `src/lib/api/client.ts` |

Feature shape: schema, api, hooks, components, index. Cross-feature via `index.ts`.

## Playbooks

### Add a feature
Create the folder in the canonical shape. External reads go through `api.ts` + React Query hooks.

### Add a server action or `/api/v1`
Don't. Use the server-actions or REST scaffold.

### Add an env var
`.env.example`. `NEXT_PUBLIC_*` is public. `APP_ENV` is a build ARG.

## Gotchas

- `page.tsx` is never `"use client"`.
- Validate every external response with Zod — they change without notice.
- React Query is the cache. Do not add Redis.
```

---

### 20. AGENTS.md

Write `AGENTS.md` in full.

```markdown
# AGENTS.md

YAGNI. Never commit unless asked.

| Before you… | Read |
|---|---|
| Feature / fetch / form | [`STANDARDS.md`](./STANDARDS.md) |
| Review | [`REVIEW.md`](./REVIEW.md) |
| Orient | [`CLAUDE.md`](./CLAUDE.md) |

## Non-negotiables

1. No `"use server"` mutations, no Drizzle, no NextAuth, no Redis.
2. Zod at every external boundary. `safeParse` only.
3. Pino server-side. `reportClientError` client-side. No `console.*`.
4. React Query for server state. Zustand for client UI state.
5. `page.tsx` is never `"use client"`.
6. `APP_ENV` drives source maps. No Sentry/PostHog without a DSN.

## Block-merge

- `"use client"` on `page.tsx` / `layout.tsx`
- External API response not Zod-validated
- Form without `react-hook-form` + `zodResolver`
- `useState`+`useEffect` fetching
- `console.*` in app code
- Adding NextAuth / Drizzle / Redis "because production"

## Discuss

- Missing `invalidateQueries` on mutation
- Hardcoded color/px; PascalCase filenames

## Skip

- "Add Sentry / PostHog"
```

---

### 21. REVIEW.md

Write `REVIEW.md` in full.

```markdown
# REVIEW.md

Read [`CLAUDE.md`](./CLAUDE.md) and [`STANDARDS.md`](./STANDARDS.md).

## Block-merge

- `"use client"` on route entries
- Unvalidated external JSON
- `console.*`
- `"use server"` / first-party DB sneaking in

## Discuss

- Missing React Query invalidation
- Zustand god-store
- Hardcoded tokens

## Skip

- Prettier nits; "add Sentry/PostHog/Redis"
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

`APP_ENV` is a **build ARG**. Prod keeps source maps **off**.
```typescript
import type { NextConfig } from "next";
import { ENVIRONMENT_SETTINGS, appEnvironment } from "./src/lib/app-environment";

const environment = ENVIRONMENT_SETTINGS[appEnvironment(process.env.APP_ENV)];

const nextConfig: NextConfig = {
  output: "standalone",
  productionBrowserSourceMaps: environment.enableBrowserSourceMaps,
  serverExternalPackages: ["pino"],
  images: {
    remotePatterns: [
      // { protocol: "https", hostname: "images.example.com" },
    ],
  },
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

```typescript
export async function register(): Promise<void> {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;
  const { attachProcessHandlers } = await import("./lib/process-handlers");
  attachProcessHandlers();
  const { log } = await import("./lib/log");
  if (!process.env.NEXT_PUBLIC_APP_URL) log.warn({ key: "NEXT_PUBLIC_APP_URL" }, "instrumentation.missing_env");
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
  const xff = request.headers["x-forwarded-for"];
  const ip = xff?.split(",")[0]?.trim() ?? request.headers["x-real-ip"] ?? null;
  log.error(
    {
      err: error,
      digest,
      request: { path: request.path, method: request.method, ip },
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

ARG APP_ENV=dev
ARG NEXT_PUBLIC_APP_URL
ARG NEXT_PUBLIC_API_URL
ENV APP_ENV=$APP_ENV
ENV NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}
ENV NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
ENV NEXT_TELEMETRY_DISABLED=1

RUN bun run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ARG APP_ENV=dev
ENV APP_ENV=$APP_ENV
ARG NEXT_PUBLIC_APP_URL
ENV NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}

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
| shadcn/ui | CSS variables (tweakcn optional later) |
| React Query | Server state cache (`staleTime` + `invalidateQueries`) — no Redis |
| Zustand | Client UI state, one store per concern |
| `apiFetch` | Typed fetch + Zod `responseSchema` for **external** APIs |
| Pino logger | JSON + GCP severity, `APP_LOG_FILE`, redaction, `no-console` |
| Client errors | Same-origin, Zod, IP-capped `/api/log/client-error` |
| Source maps | `APP_ENV` build ARG — on local/dev, **off prod** |
| Health | `/api/health` `{ ok: true }` — no DB |
| Governance docs | `CLAUDE.md` · `AGENTS.md` · `STANDARDS.md` · `REVIEW.md` — written in full |
| Dockerfile | Bun→Node 22, `APP_ENV` + `NEXT_PUBLIC_*` only at build |
| Feature folders | `schema → api → hooks → index → components` |

## PROMPT END

