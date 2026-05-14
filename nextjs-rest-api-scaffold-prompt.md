# Next.js REST API Scaffold Prompt

Copy-paste this entire prompt to scaffold a production-ready Next.js app with **strict REST APIs, zero server actions**.
Replace `{{APP_NAME}}` with your app name (kebab-case, e.g. `my-app`).

**Use this when:** the app has a frontend AND a backend, but you want every mutation to go through versioned HTTP endpoints (so a mobile client, third-party integrator, or non-TS client can consume the same API the web frontend uses). All form submits go to `/api/v1/*`; server components fetch data via direct query imports (no HTTP roundtrip for SSR); client components fetch via React Query hooks wrapping typed fetch wrappers.

**Use the full-stack server-actions prompt instead when:** the app is Next.js-only with no plan for external API consumers — server actions give you better DX with less ceremony.

**Use the frontend-only prompt instead when:** you have no first-party backend at all.

---

## PROMPT START

Scaffold a production-ready REST API + Next.js frontend app called `{{APP_NAME}}`. Follow every instruction exactly — don't add extras, don't skip steps.

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
  "next-auth": "^5.0.0-beta.30",
  "@auth/drizzle-adapter": "^1",
  "drizzle-orm": "^0.45",
  "pg": "^8",
  "zod": "^4",
  "pino": "^9",
  "clsx": "^2",
  "tailwind-merge": "^3",
  "@upstash/ratelimit": "^2",
  "@upstash/redis": "^1",
  "nodemailer": "^6",
  "react-hook-form": "^7",
  "@hookform/resolvers": "^5",
  "@tanstack/react-query": "^5",
  "@tanstack/react-query-devtools": "^5"
}
```

**Dev deps:**
```json
{
  "drizzle-kit": "^0.31",
  "vitest": "^3",
  "@vitejs/plugin-react": "^4",
  "@testing-library/react": "^16",
  "@testing-library/jest-dom": "^6",
  "jsdom": "^25",
  "pino-pretty": "^11",
  "@types/pg": "^8",
  "@types/nodemailer": "^6",
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

### 6. Database — Drizzle + PostgreSQL (NeonDB)

**Current provider: NeonDB** (serverless Postgres). If you switch providers later, only `DATABASE_URL` changes — Drizzle and the app code are provider-agnostic as long as the target is PostgreSQL.

**NeonDB setup (2 minutes):**
1. Go to [console.neon.tech](https://console.neon.tech) → New Project
2. Pick a region close to your app's hosting region
3. Copy the connection string from **Connection Details → Connection string**
4. Use the **pooled** connection string for the app, the direct one only for `drizzle-kit migrate`

Add to `.env.example`:
```bash
DATABASE_URL=postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/dbname?sslmode=require
DATABASE_URL_DIRECT=postgresql://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require
```

Create `src/db/index.ts`:
```typescript
import { Pool } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema";

const databaseUrl = process.env.DATABASE_URL;
if (!databaseUrl) throw new Error("DATABASE_URL is not set");

const pool = new Pool({
  connectionString: databaseUrl,
  ssl: databaseUrl.includes("sslmode=disable") ? false : { rejectUnauthorized: false },
});

export const db = drizzle(pool, { schema });
export * from "./schema";
```

Create `src/db/schema/index.ts` — re-export all tables from here.

Create `src/db/schema/users.ts`:
```typescript
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: text("name"),
  email: text("email").notNull().unique(),
  emailVerified: timestamp("email_verified"),
  image: text("image"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});
```

Also create the NextAuth required tables — `accounts`, `sessions`, `verification_tokens` — following the `@auth/drizzle-adapter` schema exactly (copy from the adapter docs).

Create `drizzle.config.ts`:
```typescript
import { defineConfig } from "drizzle-kit";

const migrationUrl = process.env.DATABASE_URL_DIRECT ?? process.env.DATABASE_URL!;
const isLocal = migrationUrl.includes("localhost") || migrationUrl.includes("127.0.0.1");

export default defineConfig({
  schema: "./src/db/schema/*.ts",
  out: "./drizzle",
  dialect: "postgresql",
  dbCredentials: {
    url: migrationUrl,
    ssl: isLocal ? false : { rejectUnauthorized: false },
  },
});
```

Add to `package.json` scripts:
```json
{
  "db:generate": "drizzle-kit generate",
  "db:migrate": "drizzle-kit migrate",
  "db:studio": "drizzle-kit studio"
}
```

---

### 7. Auth — NextAuth v5 + Google OAuth

**Google Cloud Console setup:**
1. Go to [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID → Application type: **Web application**
3. Add Authorized redirect URIs:
   - `http://localhost:3000/api/auth/callback/google` (dev)
   - `https://your-domain.com/api/auth/callback/google` (prod)
4. Copy **Client ID** → `AUTH_GOOGLE_ID`
5. Copy **Client Secret** → `AUTH_GOOGLE_SECRET`

**TypeScript type augmentation** — `src/types/next-auth.d.ts`:
```typescript
import type { DefaultSession } from "next-auth";

declare module "next-auth" {
  interface Session {
    user: { id: string } & DefaultSession["user"];
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    id: string;
  }
}
```

**`src/features/auth/lib/config.ts`** — edge-safe config:
```typescript
import type { NextAuthConfig } from "next-auth";
import Google from "next-auth/providers/google";

export const authConfig: NextAuthConfig = {
  providers: [
    Google({
      clientId: process.env.AUTH_GOOGLE_ID,
      clientSecret: process.env.AUTH_GOOGLE_SECRET,
      authorization: { params: { prompt: "select_account" } },
    }),
  ],
  session: { strategy: "jwt" },
  trustHost: true,
  callbacks: {
    async jwt({ token, user }) {
      if (user?.id) token.id = user.id;
      return token;
    },
    async session({ session, token }) {
      if (token.id) session.user.id = token.id;
      return session;
    },
  },
  pages: { signIn: "/login", error: "/login" },
};
```

**`src/features/auth/index.ts`:**
```typescript
import NextAuth from "next-auth";
import { DrizzleAdapter } from "@auth/drizzle-adapter";
import { db } from "@/db";
import { authConfig } from "./lib/config";

export const { handlers, auth, signIn, signOut } = NextAuth({
  ...authConfig,
  adapter: DrizzleAdapter(db),
});
```

**`src/features/auth/actions.ts`** — sign-in/sign-out (the only place `"use server"` appears in this whole scaffold):
```typescript
"use server";

import { signIn, signOut } from "@/features/auth";

export async function signInWithGoogle(redirectTo?: string) {
  await signIn("google", { redirectTo: redirectTo ?? "/dashboard" });
}

export async function signOutUser() {
  await signOut({ redirectTo: "/login" });
}
```

> **Note:** `actions.ts` is intentionally minimal here — it exists only because NextAuth's `signIn`/`signOut` flows redirect via form actions. Every other mutation in the app goes through `/api/v1/*`, not server actions.

**`src/app/api/auth/[...nextauth]/route.ts`:**
```typescript
import { handlers } from "@/features/auth";
export const { GET, POST } = handlers;
```

**`src/app/(auth)/login/page.tsx`:**
```tsx
import { redirect } from "next/navigation";
import { auth } from "@/features/auth";
import { signInWithGoogle } from "@/features/auth/actions";
import { Button } from "@/components/ui/button";

export default async function LoginPage({
  searchParams,
}: {
  searchParams: Promise<{ callbackUrl?: string; error?: string }>;
}) {
  const session = await auth();
  if (session?.user) redirect("/dashboard");

  const { callbackUrl, error } = await searchParams;

  const errorMessages: Record<string, string> = {
    OAuthAccountNotLinked: "This email is already linked to another sign-in method.",
    OAuthCallbackError: "Sign-in failed. Please try again.",
    Default: "Something went wrong. Please try again.",
  };
  const errorMessage = error ? (errorMessages[error] ?? errorMessages.Default) : null;

  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="w-full max-w-sm space-y-6 rounded-lg border bg-card p-8">
        <div className="space-y-1 text-center">
          <h1 className="text-2xl font-semibold">{{APP_NAME}}</h1>
          <p className="text-sm text-muted-foreground">Sign in to continue</p>
        </div>

        {errorMessage && (
          <p className="rounded-md border border-destructive/50 px-4 py-3 text-center text-sm text-destructive">
            {errorMessage}
          </p>
        )}

        <form action={signInWithGoogle.bind(null, callbackUrl)}>
          <Button type="submit" variant="outline" className="w-full">
            Continue with Google
          </Button>
        </form>
      </div>
    </div>
  );
}
```

**Sign-out button:**
```tsx
import { signOutUser } from "@/features/auth/actions";
import { Button } from "@/components/ui/button";

export function SignOutButton() {
  return (
    <form action={signOutUser}>
      <Button type="submit" variant="ghost">Sign out</Button>
    </form>
  );
}
```

Create `src/middleware.ts` — auth guard + IP rate limiting (rate limiter is set up in section 26):
```typescript
import NextAuth from "next-auth";
import { NextResponse, type NextRequest } from "next/server";
import { authConfig } from "@/features/auth/lib/config";
import { createRateLimiter } from "@/lib/rate-limit";
import { getIp } from "@/lib/get-ip";

const { auth } = NextAuth(authConfig);

// Broad rate limit applied to all /api/v1/* traffic.
// Per-route limiters can add tighter limits inside specific route handlers.
const apiLimiter = createRateLimiter({ windowMs: 60_000, max: 60 });

export default auth(async (req) => {
  const { pathname } = req.nextUrl;

  // Rate limit public + private API routes by IP — webhooks are exempt
  // (signature verification is their gate).
  const isApiV1 = pathname.startsWith("/api/v1/");
  if (isApiV1) {
    const result = await apiLimiter(getIp(req));
    if (!result.allowed) {
      return NextResponse.json(
        { error: { code: "rate_limited", message: "Too many requests." } },
        { status: 429, headers: { "Retry-After": String(result.retryAfter) } }
      );
    }
  }

  // Auth guard — redirect unauthenticated users to login (for UI routes)
  const isLoggedIn = !!req.auth;
  const isAuthRoute = pathname.startsWith("/api/auth");
  const isWebhook = pathname.startsWith("/api/webhooks");
  const isPublicPage = ["/login", "/terms", "/privacy"].includes(pathname);

  if (isAuthRoute || isWebhook || isPublicPage) return;

  // /api/v1 routes handle their own auth via requireAuth() — let them through.
  // Unauthenticated requests get 401 from the route handler.
  if (isApiV1) return;

  if (!isLoggedIn) {
    const loginUrl = new URL("/login", req.nextUrl.origin);
    loginUrl.searchParams.set("callbackUrl", pathname);
    return Response.redirect(loginUrl);
  }
});

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|images/).*)"],
};
```

---

### 8. Core lib files

**`src/lib/utils.ts`** — `cn()` helper:
```typescript
import { type ClassValue, clsx } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

**`src/lib/api/errors.ts`** — shared `ApiError` class (server + client both import from here):
```typescript
export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string
  ) {
    super(message);
    this.name = "ApiError";
  }
}
```

**`src/lib/api/response.ts`** — server-side response helpers used by every route handler:
```typescript
import { NextResponse } from "next/server";
import { log } from "@/lib/log";
import { ApiError } from "./errors";

export function apiOk<T>(data: T, status = 200): NextResponse {
  return NextResponse.json({ data }, { status });
}

export function apiError(err: unknown): NextResponse {
  if (err instanceof ApiError) {
    return NextResponse.json(
      { error: { code: err.code, message: err.message } },
      { status: err.status }
    );
  }
  // Unknown errors — log full detail server-side, return generic message
  log.error({ err }, "api.unhandled_error");
  return NextResponse.json(
    { error: { code: "internal_error", message: "Something went wrong." } },
    { status: 500 }
  );
}

export function firstError(issues: { message: string }[], fallback: string): string {
  return issues[0]?.message ?? fallback;
}
```

**`src/lib/api/auth.ts`** — extracts an `ApiContext` from the request session:
```typescript
import { auth } from "@/features/auth";
import { ApiError } from "./errors";

export type ApiContext = {
  userId: string;
};

/**
 * Reads the session and returns an ApiContext. Throws ApiError(401) when
 * unauthenticated — route handlers should let it bubble up to apiError().
 */
export async function requireAuth(): Promise<ApiContext> {
  const session = await auth();
  if (!session?.user?.id) {
    throw new ApiError(401, "unauthorized", "Authentication required.");
  }
  return { userId: session.user.id };
}
```

**`src/lib/api/client.ts`** — typed fetch wrapper for browser code (React Query hooks call this):
```typescript
import { z } from "zod";
import { ApiError } from "./errors";

type RequestOptions<TResponse> = {
  method?: "GET" | "POST" | "PATCH" | "PUT" | "DELETE";
  body?: unknown;
  responseSchema?: z.ZodSchema<TResponse>;
  headers?: Record<string, string>;
  signal?: AbortSignal;
};

/**
 * Typed fetch wrapper. Throws ApiError on non-2xx so React Query catches
 * it and exposes error.status / error.code to your UI. Validates the
 * response body with Zod if a schema is passed.
 */
export async function apiFetch<TResponse>(
  path: string,
  options: RequestOptions<TResponse> = {}
): Promise<TResponse> {
  const { method = "GET", body, responseSchema, headers = {}, signal } = options;

  const res = await fetch(path, {
    method,
    headers: { "content-type": "application/json", ...headers },
    body: body !== undefined ? JSON.stringify(body) : undefined,
    signal,
  });

  if (!res.ok) {
    const errBody = await res.json().catch(() => ({}));
    throw new ApiError(
      res.status,
      errBody?.error?.code ?? "unknown_error",
      errBody?.error?.message ?? `Request failed: ${res.status}`
    );
  }

  if (res.status === 204) return undefined as TResponse;

  const json = await res.json();
  const data = json?.data ?? json; // unwrap { data } envelope

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

**`src/lib/log.ts`:**
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
    paths: [
      "password", "*.password", "token", "*.token", "accessToken", "*.accessToken",
      "refreshToken", "*.refreshToken", "apiKey", "*.apiKey", "secret", "*.secret",
      "authorization", "*.authorization", "headers.authorization", "headers.cookie",
      "privateKey", "*.privateKey", "credentials", "*.credentials",
    ],
    censor: "[REDACTED]",
  },
};

const devTransport: LoggerOptions["transport"] =
  !isProd && !isTest
    ? { target: "pino-pretty", options: { colorize: true, translateTime: "SYS:HH:MM:ss.l", ignore: "pid,hostname,service,severity" } }
    : undefined;

export const log: Logger = pino({ ...baseOptions, ...(devTransport ? { transport: devTransport } : {}) });

export function getClientIp(headers: Headers): string | null {
  const xff = headers.get("x-forwarded-for");
  return xff?.split(",")[0]?.trim() || headers.get("x-real-ip") || null;
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

**`src/lib/audit.ts`:**
```typescript
import { headers } from "next/headers";
import { db } from "@/db";
import { auditLogs } from "@/db/schema";
import { getClientIp } from "@/lib/log";

export type AuditAction =
  | "auth.login"
  | "auth.logout"
  | "user.updated"
  | "user.deleted";
  // Extend per feature — e.g. "post.created", "comment.deleted"

type LogAuditParams = {
  userId: string | null;
  action: AuditAction;
  resource: string;
  metadata?: Record<string, unknown>;
};

async function readRequestMeta(): Promise<{ ip: string | null; userAgent: string | null }> {
  try {
    const h = await headers();
    return { ip: getClientIp(h), userAgent: h.get("user-agent") };
  } catch {
    return { ip: null, userAgent: null };
  }
}

export async function logAudit(params: LogAuditParams): Promise<void> {
  const meta = await readRequestMeta();
  await db.insert(auditLogs).values({
    userId: params.userId,
    action: params.action,
    resource: params.resource,
    metadata: params.metadata,
    ipAddress: meta.ip,
    userAgent: meta.userAgent,
  });
}
```

Create the audit table at `src/db/schema/audit-logs.ts`:
```typescript
import { jsonb, pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { users } from "./users";

export const auditLogs = pgTable("audit_logs", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").references(() => users.id, { onDelete: "set null" }),
  action: text("action").notNull(),
  resource: text("resource").notNull(),
  metadata: jsonb("metadata"),
  ipAddress: text("ip_address"),
  userAgent: text("user_agent"),
  createdAt: timestamp("created_at").notNull().defaultNow(),
});
```

---

### 9. Zod conventions

Zod is the **only** validation library. Use it at every system boundary — API request bodies, response shapes, webhook bodies, env vars.

```typescript
// ✅ Always safeParse — never .parse() which throws
const parsed = CreateThingSchema.safeParse(input);
if (!parsed.success) {
  throw new ApiError(400, "validation_error", firstError(parsed.error.issues, "Invalid input"));
}

// ✅ Every string field gets .trim() + .min(1) + .max() with a custom user-facing message
const NameSchema = z
  .string({ message: "Name is required" })
  .trim()
  .min(1, { message: "Name is required" })
  .max(120, { message: "Name must be 120 characters or fewer" });

// ✅ Always export BOTH request input schemas AND response shape schemas
export const CreateThingInputSchema = z.object({
  name: NameSchema,
  description: z.string().max(2_000).nullable().optional(),
});

export const ThingSchema = z.object({
  id: z.string().uuid(),
  name: z.string(),
  description: z.string().nullable(),
  createdAt: z.string().datetime(),
});

export const ThingsResponseSchema = z.array(ThingSchema);

export type CreateThingInput = z.infer<typeof CreateThingInputSchema>;
export type Thing = z.infer<typeof ThingSchema>;
```

**The two-side schema pattern is the foundation of REST consistency:**
- **Server route handler** validates the incoming request body with `CreateThingInputSchema.safeParse(body)` → 400 on failure
- **Client form** uses the same `CreateThingInputSchema` as its `zodResolver` → renders field-level errors before the request leaves the browser
- **Server route handler's response** is whatever the handler returns (typed via the shared `Thing` type)
- **Client `apiFetch`** validates the response body against `ThingSchema` → catches API drift at the boundary

Same schema file is the single source of truth for both ends.

**Where Zod runs:**

| Boundary | Use Zod? |
|---|---|
| API route handler request body | ✅ Always — first line of every POST/PATCH |
| API route handler response | ✅ Always (returned shape matches schema) |
| Client form input | ✅ Always — `zodResolver` |
| Client `apiFetch` response | ✅ Always — `responseSchema` |
| Webhook body (after signature check) | ✅ Always |
| Env var validation | ✅ At first use |
| Internal function calls | ❌ Trust TypeScript |
| DB query results via Drizzle | ❌ Drizzle types it |

---

### 10. Logger conventions

Pino is the **only** logger. `console.log` is banned server-side.

**Always: context object first, message string second**
```typescript
log.info({ userId, thingId: row.id }, "things.created");
log.error({ userId, route: "/api/v1/things", err }, "things.create_failed");

// ❌ String interpolation kills structured search
log.info(`Created thing ${row.id}`);
```

**Event name format: `feature.verb`**
```
"auth.login"          "things.created"          "stripe.webhook_received"
```

**Log levels:**

| Level | When | Always include |
|---|---|---|
| `log.fatal` | Process is going down | `err` |
| `log.error` | Request failed | `err` + context |
| `log.warn` | Recoverable degradation | context |
| `log.info` | State change worth audit | `userId` |
| `log.debug` | Dev-only | anything |

**Never log sensitive values** — even at debug. Log shapes, not values:
```typescript
// ❌
log.info({ apiKey: input.apiKey }, "integration.key_set");

// ✅
log.info({ userId, provider, keyPrefix: key.slice(0, 4) }, "integration.key_set");
```

**Child logger for request-scoped context:**
```typescript
const reqLog = log.child({ requestId, userId });
reqLog.info("things.created");
```

**Client-side errors** — use `reportClientError()`:
```tsx
import { toast } from "sonner";
import { reportClientError } from "@/lib/report-client-error";

try {
  await mutation.mutateAsync(input);
} catch (err) {
  reportClientError(err, { feature: "things" });
  toast.error("Something went wrong.");
}
```

Create `src/lib/report-client-error.ts`:
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

Create `src/app/api/log/client-error/route.ts`:
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

### 11. Feature folder pattern

Create `src/features/_template/` — the canonical feature shape every other feature copies:

```
src/features/<name>/
  schema.ts       ← Zod schemas — request inputs + response shapes
  queries.ts      ← DB reads (used by server components AND handlers)
  mutations.ts    ← DB writes (used by handlers only)
  handlers.ts     ← Pure business logic: (ctx, input) → output. NO HTTP, NO auth, NO request parsing
  api.ts          ← Client-side typed fetch wrappers — calls /api/v1/<resource>
  hooks/          ← React Query hooks wrapping api.ts (one file per query / mutation)
  index.ts        ← Public barrel
  components/     ← Server + client components for this feature
```

And the HTTP routes:
```
src/app/api/v1/<resource>/
  route.ts        ← Thin HTTP adapter: parse body → requireAuth → safeParse → call handler → respond
  [id]/route.ts   ← GET, PATCH, DELETE for a single resource
```

**The split — why `handlers.ts` and `route.ts` are separate files:**

| File | Responsibility | Imports |
|---|---|---|
| `handlers.ts` | Pure business logic. Takes `(ctx, input)`, returns data or throws `ApiError`. No `Request`, no parsing, no auth — caller's job. | DB queries, audit, log |
| `route.ts` | HTTP adapter. Parses body, calls `requireAuth(req)`, validates with Zod, calls handler, formats response. | `requireAuth`, schemas, `handlers` |

This split means:
- Handlers are unit-testable without HTTP — pass a mock `ctx`, assert the return value
- Multiple HTTP routes can call the same handler (e.g. `POST /api/v1/things` and `POST /api/v1/admin/things` both call `createThing`)
- Server components that need to render the same data import from `queries.ts` directly — no HTTP roundtrip for SSR

---

#### Canonical feature files

**`schema.ts`** — both request and response schemas:
```typescript
import { z } from "zod";

const NAME_MAX = 120;
const DESCRIPTION_MAX = 2_000;

const UuidSchema = z.string().uuid({ message: "Invalid id" });

export const ThingNameSchema = z
  .string({ message: "Name is required" })
  .trim()
  .min(1, { message: "Name is required" })
  .max(NAME_MAX, { message: `Name must be ${NAME_MAX} characters or fewer` });

// ── Request inputs ─────────────────────────────────────────────────────────

export const CreateThingInputSchema = z.object({
  name: ThingNameSchema,
  description: z
    .string()
    .max(DESCRIPTION_MAX, { message: `Description must be ${DESCRIPTION_MAX} characters or fewer` })
    .nullable()
    .optional(),
});

export const UpdateThingInputSchema = z.object({
  name: ThingNameSchema.optional(),
  description: z.string().max(DESCRIPTION_MAX).nullable().optional(),
});

// ── Response shapes ────────────────────────────────────────────────────────

export const ThingSchema = z.object({
  id: UuidSchema,
  name: z.string(),
  description: z.string().nullable(),
  createdAt: z.string().datetime(),
  updatedAt: z.string().datetime(),
});

export const ThingsResponseSchema = z.array(ThingSchema);

// ── Inferred types ─────────────────────────────────────────────────────────

export type CreateThingInput = z.infer<typeof CreateThingInputSchema>;
export type UpdateThingInput = z.infer<typeof UpdateThingInputSchema>;
export type Thing = z.infer<typeof ThingSchema>;
```

**`queries.ts`** — DB reads:
```typescript
import { and, eq } from "drizzle-orm";
import { db } from "@/db";
import { things } from "@/db/schema";
import type { Thing } from "./schema";

type ThingRow = typeof things.$inferSelect;

function toThing(row: ThingRow): Thing {
  return {
    id: row.id,
    name: row.name,
    description: row.description,
    createdAt: row.createdAt.toISOString(),
    updatedAt: row.updatedAt.toISOString(),
  };
}

export async function listUserThings(userId: string): Promise<Thing[]> {
  const rows = await db.query.things.findMany({
    where: eq(things.userId, userId),
    orderBy: (t, { desc }) => [desc(t.createdAt)],
  });
  return rows.map(toThing);
}

export async function findUserThing(userId: string, id: string): Promise<Thing | null> {
  const row = await db.query.things.findFirst({
    where: and(eq(things.id, id), eq(things.userId, userId)),
  });
  return row ? toThing(row) : null;
}
```

**`mutations.ts`** — DB writes (no `"use server"`):
```typescript
import { and, eq } from "drizzle-orm";
import { db } from "@/db";
import { things } from "@/db/schema";
import type { CreateThingInput, UpdateThingInput, Thing } from "./schema";
import { findUserThing } from "./queries";

export async function insertThing(userId: string, input: CreateThingInput): Promise<Thing | null> {
  const [row] = await db
    .insert(things)
    .values({
      userId,
      name: input.name,
      description: input.description ?? null,
    })
    .returning();
  if (!row) return null;
  return findUserThing(userId, row.id);
}

export async function updateThing(
  userId: string,
  id: string,
  fields: UpdateThingInput
): Promise<Thing | null> {
  await db
    .update(things)
    .set({ ...fields, updatedAt: new Date() })
    .where(and(eq(things.id, id), eq(things.userId, userId)));
  return findUserThing(userId, id);
}

export async function deleteThing(userId: string, id: string): Promise<void> {
  await db.delete(things).where(and(eq(things.id, id), eq(things.userId, userId)));
}
```

**`handlers.ts`** — pure business logic. Receives `ApiContext`, returns data or throws `ApiError`. Does NOT touch `Request`, `NextResponse`, or HTTP concerns:
```typescript
import type { ApiContext } from "@/lib/api/auth";
import { ApiError } from "@/lib/api/errors";
import { log } from "@/lib/log";
import { logAudit } from "@/lib/audit";
import { findUserThing, listUserThings } from "./queries";
import { insertThing, updateThing, deleteThing } from "./mutations";
import type { CreateThingInput, UpdateThingInput, Thing } from "./schema";

export async function listThings(ctx: ApiContext): Promise<Thing[]> {
  return listUserThings(ctx.userId);
}

export async function getThing(ctx: ApiContext, id: string): Promise<Thing> {
  const thing = await findUserThing(ctx.userId, id);
  if (!thing) throw new ApiError(404, "not_found", "Thing not found.");
  return thing;
}

export async function createThing(ctx: ApiContext, input: CreateThingInput): Promise<Thing> {
  const thing = await insertThing(ctx.userId, input);
  if (!thing) throw new ApiError(500, "internal_error", "Could not create thing.");

  log.info({ userId: ctx.userId, thingId: thing.id }, "things.created");
  await logAudit({
    userId: ctx.userId,
    action: "thing.created",
    resource: thing.id,
    metadata: { name: input.name },
  });

  return thing;
}

export async function patchThing(
  ctx: ApiContext,
  id: string,
  input: UpdateThingInput
): Promise<Thing> {
  // Verify ownership before update — surface 404 not 500
  const existing = await findUserThing(ctx.userId, id);
  if (!existing) throw new ApiError(404, "not_found", "Thing not found.");

  const updated = await updateThing(ctx.userId, id, input);
  if (!updated) throw new ApiError(500, "internal_error", "Could not update thing.");

  log.info({ userId: ctx.userId, thingId: id }, "things.updated");
  return updated;
}

export async function removeThing(ctx: ApiContext, id: string): Promise<void> {
  const existing = await findUserThing(ctx.userId, id);
  if (!existing) throw new ApiError(404, "not_found", "Thing not found.");

  await deleteThing(ctx.userId, id);

  log.info({ userId: ctx.userId, thingId: id }, "things.deleted");
  await logAudit({ userId: ctx.userId, action: "thing.deleted", resource: id });
}
```

**`src/app/api/v1/things/route.ts`** — collection endpoint (list, create):
```typescript
import type { NextRequest } from "next/server";
import { apiOk, apiError, firstError } from "@/lib/api/response";
import { ApiError } from "@/lib/api/errors";
import { requireAuth } from "@/lib/api/auth";
import { listThings, createThing } from "@/features/things/handlers";
import { CreateThingInputSchema } from "@/features/things/schema";

export const runtime = "nodejs";

export async function GET() {
  try {
    const ctx = await requireAuth();
    const things = await listThings(ctx);
    return apiOk(things);
  } catch (err) {
    return apiError(err);
  }
}

export async function POST(req: NextRequest) {
  try {
    const ctx = await requireAuth();
    const body = await req.json().catch(() => {
      throw new ApiError(400, "bad_request", "Invalid JSON.");
    });
    const parsed = CreateThingInputSchema.safeParse(body);
    if (!parsed.success) {
      throw new ApiError(400, "validation_error", firstError(parsed.error.issues, "Invalid input"));
    }
    const thing = await createThing(ctx, parsed.data);
    return apiOk(thing, 201);
  } catch (err) {
    return apiError(err);
  }
}
```

**`src/app/api/v1/things/[id]/route.ts`** — single-resource endpoint:
```typescript
import type { NextRequest } from "next/server";
import { apiOk, apiError, firstError } from "@/lib/api/response";
import { ApiError } from "@/lib/api/errors";
import { requireAuth } from "@/lib/api/auth";
import { getThing, patchThing, removeThing } from "@/features/things/handlers";
import { UpdateThingInputSchema } from "@/features/things/schema";

export const runtime = "nodejs";

type Context = { params: Promise<{ id: string }> };

export async function GET(_req: NextRequest, { params }: Context) {
  try {
    const ctx = await requireAuth();
    const { id } = await params;
    const thing = await getThing(ctx, id);
    return apiOk(thing);
  } catch (err) {
    return apiError(err);
  }
}

export async function PATCH(req: NextRequest, { params }: Context) {
  try {
    const ctx = await requireAuth();
    const { id } = await params;
    const body = await req.json().catch(() => {
      throw new ApiError(400, "bad_request", "Invalid JSON.");
    });
    const parsed = UpdateThingInputSchema.safeParse(body);
    if (!parsed.success) {
      throw new ApiError(400, "validation_error", firstError(parsed.error.issues, "Invalid input"));
    }
    const thing = await patchThing(ctx, id, parsed.data);
    return apiOk(thing);
  } catch (err) {
    return apiError(err);
  }
}

export async function DELETE(_req: NextRequest, { params }: Context) {
  try {
    const ctx = await requireAuth();
    const { id } = await params;
    await removeThing(ctx, id);
    return new Response(null, { status: 204 });
  } catch (err) {
    return apiError(err);
  }
}
```

**`api.ts`** — client-side fetch wrappers (used by React Query hooks):
```typescript
import { apiFetch } from "@/lib/api/client";
import {
  ThingSchema,
  ThingsResponseSchema,
  type CreateThingInput,
  type UpdateThingInput,
  type Thing,
} from "./schema";

export function listThings(): Promise<Thing[]> {
  return apiFetch("/api/v1/things", { responseSchema: ThingsResponseSchema });
}

export function getThing(id: string): Promise<Thing> {
  return apiFetch(`/api/v1/things/${id}`, { responseSchema: ThingSchema });
}

export function createThing(input: CreateThingInput): Promise<Thing> {
  return apiFetch("/api/v1/things", {
    method: "POST",
    body: input,
    responseSchema: ThingSchema,
  });
}

export function updateThing(id: string, input: UpdateThingInput): Promise<Thing> {
  return apiFetch(`/api/v1/things/${id}`, {
    method: "PATCH",
    body: input,
    responseSchema: ThingSchema,
  });
}

export function deleteThing(id: string): Promise<void> {
  return apiFetch(`/api/v1/things/${id}`, { method: "DELETE" });
}
```

**`hooks/use-things.ts`** — React Query hooks. One file per query/mutation:
```typescript
"use client";

import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { listThings, createThing, updateThing, deleteThing } from "../api";
import type { CreateThingInput, UpdateThingInput } from "../schema";

const QUERY_KEY = ["things"];

export function useThings() {
  return useQuery({ queryKey: QUERY_KEY, queryFn: listThings });
}

export function useCreateThing() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: CreateThingInput) => createThing(input),
    onSuccess: () => qc.invalidateQueries({ queryKey: QUERY_KEY }),
  });
}

export function useUpdateThing(id: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (input: UpdateThingInput) => updateThing(id, input),
    onSuccess: () => qc.invalidateQueries({ queryKey: QUERY_KEY }),
  });
}

export function useDeleteThing() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (id: string) => deleteThing(id),
    onSuccess: () => qc.invalidateQueries({ queryKey: QUERY_KEY }),
  });
}
```

**`index.ts`** — public barrel:
```typescript
export { ThingsListPage } from "./components/things-list-page";
export { ThingDetailPage } from "./components/thing-detail-page";

export { useThings, useCreateThing, useUpdateThing, useDeleteThing } from "./hooks/use-things";

export type { CreateThingInput, UpdateThingInput, Thing } from "./schema";
```

---

#### HTTP status code conventions

Every handler throws `ApiError(status, code, message)` for non-2xx outcomes. Use these:

| Status | When |
|---|---|
| 200 | GET success, PATCH/PUT success returning the updated resource |
| 201 | POST success that created a resource |
| 204 | DELETE success (no body) |
| 400 | Bad request — malformed JSON, validation failure |
| 401 | No auth or invalid auth |
| 403 | Authenticated but not authorized for this resource |
| 404 | Resource doesn't exist or is scoped to a different user |
| 409 | Conflict (duplicate unique key, optimistic concurrency mismatch) |
| 422 | Business rule violation (quota exceeded, state machine transition denied) |
| 429 | Rate limited (returned by middleware, not by handlers) |
| 500 | Server error (unhandled exception, DB write failed) |

---

#### Response envelope

**Success** (any 2xx with a body):
```json
{ "data": { "id": "...", "name": "..." } }
```

**Error** (any 4xx or 5xx):
```json
{ "error": { "code": "validation_error", "message": "Name is required" } }
```

The wrapper is consistent across every endpoint so consumers can write one error handler.

---

#### Forms — react-hook-form + Zod + useMutation

**Every form uses `react-hook-form` + `zodResolver`** with the same Zod schema the API route validates. Submission goes through a React Query `useMutation` hook (not a server action). No exceptions.

```tsx
// src/features/things/components/create-thing-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { toast } from "sonner";
import { ApiError } from "@/lib/api/errors";
import { CreateThingInputSchema, type CreateThingInput } from "../schema";
import { useCreateThing } from "../hooks/use-things";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  Form, FormControl, FormField, FormItem, FormLabel, FormMessage,
} from "@/components/ui/form";

export function CreateThingForm() {
  const form = useForm<CreateThingInput>({
    resolver: zodResolver(CreateThingInputSchema),
    defaultValues: { name: "", description: "" },
  });

  const mutation = useCreateThing();

  async function onSubmit(values: CreateThingInput) {
    try {
      await mutation.mutateAsync(values);
      toast.success("Thing created");
      form.reset();
    } catch (err) {
      if (err instanceof ApiError) {
        form.setError("root", { message: err.message });
      } else {
        form.setError("root", { message: "Something went wrong. Please try again." });
      }
    }
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
          name="description"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Description</FormLabel>
              <FormControl><Textarea {...field} value={field.value ?? ""} /></FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {form.formState.errors.root && (
          <p className="text-sm text-destructive">{form.formState.errors.root.message}</p>
        )}

        <Button type="submit" disabled={mutation.isPending}>
          {mutation.isPending ? "Creating..." : "Create"}
        </Button>
      </form>
    </Form>
  );
}
```

**Rules — enforce all of these:**

- **One schema, two consumers.** `CreateThingInputSchema` lives in `feature/schema.ts`. The route handler validates with `.safeParse()`. The form validates with `zodResolver()`. Same source of truth.
- **`onSubmit` calls `mutation.mutateAsync()`** — let React Query handle cache invalidation via `onSuccess`. Don't manually patch state.
- **`ApiError` from the server → `form.setError("root", ...)`** — surface the server's error message on the form. Field-level Zod errors render automatically via `<FormMessage />`.
- **Disable submit with `mutation.isPending`** — React Query exposes this; use it, not `form.formState.isSubmitting` (the latter only covers the form-level state, not the underlying request).
- **For optional nullable fields**, controlled inputs need `value={field.value ?? ""}`.

---

#### Feature scaffolder script

Create `scripts/create-feature.ts`:
```typescript
#!/usr/bin/env bun
//
// Scaffolds a new feature from src/features/_template/.
//
//   bun run create-feature <kebab-name>
//   bun run create-feature billing-events
//   bun run create-feature people --singular Person

import * as fs from "node:fs";
import * as path from "node:path";
import { fileURLToPath } from "node:url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const ROOT = path.resolve(__dirname, "..");

const args = process.argv.slice(2);
const flags: Record<string, string | true> = {};
const positionals: string[] = [];

for (let i = 0; i < args.length; i++) {
  const a = args[i];
  if (a.startsWith("--")) {
    const key = a.slice(2);
    const next = args[i + 1];
    if (next && !next.startsWith("--")) { flags[key] = next; i++; }
    else { flags[key] = true; }
  } else { positionals.push(a); }
}

const name = positionals[0];
if (!name || flags.help === true) {
  printUsage();
  process.exit(name ? 0 : 1);
}

if (!/^[a-z][a-z0-9-]*[a-z0-9]$/.test(name)) {
  fail(`Invalid feature name "${name}". Use kebab-case.`);
}
if (name === "_template") fail(`The name "_template" is reserved.`);

const RESERVED = new Set(["app", "lib", "test", "components", "auth", "db", "types"]);
if (RESERVED.has(name)) fail(`The name "${name}" is reserved.`);

const templateDir = path.join(ROOT, "src/features/_template");
const targetDir = path.join(ROOT, `src/features/${name}`);

if (!fs.existsSync(templateDir)) fail(`Template not found at ${templateDir}.`);
if (fs.existsSync(targetDir)) fail(`Feature already exists at ${targetDir}.`);

const pascalPlural = name
  .split("-")
  .map((s) => s.charAt(0).toUpperCase() + s.slice(1))
  .join("");

const pascalSingularDefault = pascalPlural.replace(/s$/, "") || pascalPlural;
const pascalSingular = typeof flags.singular === "string" ? flags.singular : pascalSingularDefault;
const kebabSingular = pascalSingular.replace(/([a-z])([A-Z])/g, "$1-$2").toLowerCase();

const replacements: [RegExp, string][] = [
  [/_template/g, name],
  [/Things/g, pascalPlural],
  [/Thing/g, pascalSingular],
  [/things/g, name],
  [/thing/g, kebabSingular],
];

function applyReplacements(input: string): string {
  let out = input;
  for (const [pattern, replacement] of replacements) {
    out = out.replace(pattern, replacement);
  }
  return out;
}

let filesWritten = 0;

function copyTree(srcDir: string, dstDir: string): void {
  fs.mkdirSync(dstDir, { recursive: true });
  for (const entry of fs.readdirSync(srcDir, { withFileTypes: true })) {
    const srcPath = path.join(srcDir, entry.name);
    const newName = applyReplacements(entry.name);
    const dstPath = path.join(dstDir, newName);
    if (entry.isDirectory()) copyTree(srcPath, dstPath);
    else {
      const content = fs.readFileSync(srcPath, "utf8");
      fs.writeFileSync(dstPath, applyReplacements(content));
      filesWritten++;
    }
  }
}

copyTree(templateDir, targetDir);

const relTarget = path.relative(ROOT, targetDir);
console.log(`Created ${relTarget}/  (${filesWritten} files)`);
console.log("");
console.log("Next steps:");
console.log(`  1. Define the DB table in src/db/schema/${name}.ts and re-export from src/db/schema/index.ts`);
console.log(`  2. Add AuditAction values to src/lib/audit.ts (e.g. "${kebabSingular}.created")`);
console.log(`  3. Create routes at src/app/api/v1/${name}/route.ts and [id]/route.ts`);
console.log(`  4. Build out components under src/features/${name}/components/`);
console.log(`  5. bun run db:generate && bun run db:migrate`);
console.log(`  6. bunx tsc --noEmit && bunx vitest run`);

function fail(message: string): never {
  process.stderr.write(`error: ${message}\n`);
  process.exit(1);
}

function printUsage(): void {
  console.log("Usage: bun run create-feature <kebab-name> [--singular <PascalNoun>]");
}
```

Wire it up:
```json
{ "scripts": { "create-feature": "bun scripts/create-feature.ts" } }
```

---

### 12. State management — React Query

Server state goes through React Query, not `useState`/`useEffect`. The hooks pattern is shown above in section 11; this section sets up the provider.

**`src/components/providers.tsx`:**
```tsx
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
            staleTime: 60 * 1000,
            refetchOnWindowFocus: false,
            retry: (failureCount, error: unknown) => {
              // Don't retry 4xx errors — they're not transient
              if (
                typeof error === "object" &&
                error !== null &&
                "status" in error &&
                typeof error.status === "number" &&
                error.status >= 400 &&
                error.status < 500
              ) {
                return false;
              }
              return failureCount < 1;
            },
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

**Rules:**
- **One `QueryClient` per app** — created with `useState` inside the client provider. Without `useState` the client recreates on every render and you lose the cache.
- **`staleTime: 60 * 1000`** default — without this, every component re-render triggers a refetch.
- **Don't retry 4xx errors** — they're auth/validation/not-found problems, not transient. The retry callback above filters them out.
- **Query keys are arrays, not strings.** `["things", { filter: "active" }]` not `"things-active"`.
- **`onSuccess` in mutations invalidates related queries** — let React Query handle cache updates, don't manually patch.
- **Server components don't use React Query** — they import from `queries.ts` directly. Hydrate the React Query cache from the server-rendered data if you need both.

---

### 13. App structure

```
src/
  app/
    (auth)/
      login/
        page.tsx
    (platform)/
      layout.tsx           ← auth gate + app shell
      dashboard/
        page.tsx
    api/
      auth/[...nextauth]/route.ts
      health/route.ts
      log/client-error/route.ts
      v1/
        things/
          route.ts
          [id]/route.ts
      webhooks/
        stripe/route.ts
    error.tsx
    global-error.tsx
    not-found.tsx
  features/
    auth/
      lib/config.ts
      actions.ts           ← ONLY for signIn/signOut — every other mutation is via /api/v1/*
      index.ts
    _template/
      schema.ts
      queries.ts
      mutations.ts
      handlers.ts
      api.ts
      hooks/
      index.ts
      components/
  lib/
    api/
      errors.ts
      response.ts          ← apiOk, apiError, firstError
      auth.ts              ← requireAuth, ApiContext
      client.ts            ← apiFetch
    log.ts
    audit.ts
    get-ip.ts
    rate-limit.ts
    report-client-error.ts
    utils.ts
  components/
    providers.tsx          ← React Query provider
    ui/                    ← shadcn components
  db/
    index.ts
    schema/
      index.ts
      users.ts
      audit-logs.ts
  types/
    next-auth.d.ts
  test/
    mocks/
      auth.ts
      db.ts
  middleware.ts
  instrumentation.ts
```

**`src/app/(platform)/layout.tsx`:**
```typescript
import { redirect } from "next/navigation";
import { auth } from "@/features/auth";

export default async function PlatformLayout({ children }: { children: React.ReactNode }) {
  const session = await auth();
  if (!session?.user) redirect("/login");
  return <>{children}</>;
}
```

**`src/app/api/health/route.ts`:**
```typescript
import { NextResponse } from "next/server";
import { db } from "@/db";
import { sql } from "drizzle-orm";

export const runtime = "nodejs";

export async function GET() {
  try {
    await db.execute(sql`SELECT 1`);
    return NextResponse.json({ ok: true });
  } catch {
    return NextResponse.json({ ok: false }, { status: 503 });
  }
}
```

---

### 14. Vitest

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
      DATABASE_URL: "postgresql://test:test@localhost/test",
      NEXT_PUBLIC_APP_URL: "https://app.test",
      AUTH_SECRET: "test-secret-at-least-32-chars-long!!",
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

afterEach(() => cleanup());
```

Create `src/test/mocks/auth.ts`:
```typescript
import type { Session } from "next-auth";
import type { ApiContext } from "@/lib/api/auth";

const DEFAULT_USER = {
  id: "user-1",
  email: "test@example.com",
  name: "Test User",
};

export function buildSession(overrides?: Partial<typeof DEFAULT_USER>): Session {
  return {
    user: { ...DEFAULT_USER, ...overrides },
    expires: new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString(),
  } as Session;
}

export function buildApiContext(overrides?: Partial<ApiContext>): ApiContext {
  return { userId: "user-1", ...overrides };
}
```

Create `src/test/mocks/db.ts` — chainable Drizzle mock builders:
```typescript
import { vi } from "vitest";

export function buildSelectChain<T = unknown>(resolved: T[] = []) {
  const chain = {
    from: vi.fn(), where: vi.fn().mockResolvedValue(resolved),
    innerJoin: vi.fn(), leftJoin: vi.fn(), groupBy: vi.fn(), orderBy: vi.fn(),
  };
  chain.from.mockReturnValue(chain);
  chain.innerJoin.mockReturnValue(chain);
  chain.leftJoin.mockReturnValue(chain);
  chain.groupBy.mockReturnValue(chain);
  chain.orderBy.mockReturnValue(chain);
  return chain;
}

export function buildUpdateChain<T = unknown>(resolved: T[] = []) {
  const chain = {
    set: vi.fn(), where: vi.fn().mockResolvedValue(resolved),
    returning: vi.fn().mockResolvedValue(resolved),
  };
  chain.set.mockReturnValue(chain);
  chain.where.mockReturnValue(chain);
  return chain;
}

export function buildInsertChain<T = unknown>(resolved: T[] = []) {
  const upsertChain = {
    returning: vi.fn().mockResolvedValue(resolved),
    then(onFulfilled?: ((v: T[]) => unknown) | null, onRejected?: ((r: unknown) => unknown) | null) {
      return Promise.resolve(resolved).then(onFulfilled as never, onRejected);
    },
  };
  const chain = {
    values: vi.fn(), returning: vi.fn().mockResolvedValue(resolved),
    onConflictDoNothing: vi.fn().mockResolvedValue(resolved),
    onConflictDoUpdate: vi.fn().mockReturnValue(upsertChain),
  };
  chain.values.mockReturnValue(chain);
  return chain;
}

export function buildDeleteChain() {
  return { where: vi.fn().mockResolvedValue(undefined) };
}
```

Add to `package.json`:
```json
{ "test": "vitest run" }
```

**Test patterns:**
- **Handlers** — call directly with `buildApiContext()`, assert returned data or thrown `ApiError`. No HTTP needed
- **Routes** — integration tests that mock auth + DB but exercise the full request → response cycle
- **Hooks** — wrap in `QueryClientProvider`, use `renderHook` from `@testing-library/react`
- **Schemas** — direct `Schema.safeParse(...)` assertions

**Minimum 3 cases per handler:** happy path, auth-fail (handler called with no valid ctx would never happen — the route layer guards that), validation-fail (handler invoked with input that should reject — usually only matters when the handler does business-rule validation beyond Zod).

---

### 15. Environment variables

Create `.env.example`:
```bash
# Database (NeonDB)
DATABASE_URL=postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/dbname?sslmode=require
DATABASE_URL_DIRECT=postgresql://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require

# Auth
AUTH_SECRET=generate-with-openssl-rand-base64-32
AUTH_GOOGLE_ID=your-google-oauth-client-id
AUTH_GOOGLE_SECRET=your-google-oauth-client-secret

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
LOG_LEVEL=debug

# Upstash Redis (rate limiting)
UPSTASH_REDIS_REST_URL=https://your-db.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-token

# Email (Gmail SMTP via App Password)
GMAIL_USER=your-app@gmail.com
GMAIL_APP_PASSWORD=xxxxxxxxxxxxxxxx
EMAIL_FROM_NAME={{APP_NAME}}
```

Add to `.gitignore`:
```
.env.local
.env.production
.env*.local
```

---

### 16. TypeScript config

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

### 17. Styling conventions

**Component file naming — always kebab-case:**
```
✅ home-navbar.tsx     ✅ user-avatar.tsx
❌ HomeNavbar.tsx      ❌ UserAvatar.tsx
```

The exported component name is PascalCase:
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

// ✅ Semantic tokens
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
<div className="w-full max-w-5xl">         // constrained but fluid
<div className="h-screen">                 // viewport-relative
<div className="w-full md:w-1/2 lg:w-1/3"> // responsive columns
```

**Responsive design — mobile-first:**
```tsx
<div className="flex flex-col gap-4 md:flex-row md:gap-6">
<div className="text-base md:text-lg lg:text-xl">
```

---

**`page.tsx` files never use `"use client"`:**
```tsx
// ❌ Kills SSR — crawler sees an empty shell
"use client";
export default function ProductPage() { /* ... */ }

// ✅ Page stays server-rendered. Interactivity goes in a child marked "use client"
import { listUserThings } from "@/features/things/queries";
import { AddThingButton } from "./_components/add-thing-button";

export default async function ThingsPage() {
  const session = await auth();
  const things = await listUserThings(session!.user.id); // server-side read
  return (
    <>
      <ul>{things.map((t) => <li key={t.id}>{t.name}</li>)}</ul>
      <AddThingButton />
    </>
  );
}
```

**Rule of thumb:** if the file is `page.tsx`, `layout.tsx`, or any route entry, it stays a server component. Push interactivity into child client components.

---

### 18. STANDARDS.md

Create `STANDARDS.md`:

```markdown
# STANDARDS.md

Engineering rules for this repo. Every code change should satisfy them.

1. **No `any`, no `as` casts on user input** — validate with Zod at every boundary
2. **`safeParse` everywhere, `firstError` for the user message** — never `.parse()`
3. **Every route handler returns `apiOk(data)` or throws `ApiError`** — never raw `NextResponse.json` for success/error responses
4. **`requireAuth()` is the first line of every protected route handler** — before any DB call
5. **`handlers.ts` is HTTP-agnostic** — never imports `Request`, `NextResponse`, or `headers()`. Pure `(ctx, input) → output` shape
6. **No `console.log` server-side** — use `log` from `@/lib/log`
7. **`log.info` + `logAudit` after every state change** — state changes have an audit trail
8. **No top-level `throw` for runtime env vars** — lazy getter pattern only
9. **`"use server"` only in `features/auth/actions.ts`** — every other mutation goes through `/api/v1/*`
10. **API responses follow the envelope** — success: `{ data: ... }`, error: `{ error: { code, message } }`
11. **API routes live under `/api/v1/`** — new breaking changes go to `/api/v2/`, never edit a published version
12. **HTTP status codes follow the convention table** — 201 for POST that creates, 204 for DELETE, 404 for scoped-out resources
13. **`apiFetch({ responseSchema })` validates every API response client-side** — even your own API, even in dev
14. **Server state goes through React Query, not `useState`/`useEffect`** — `useState` fetching causes race conditions
15. **Commit messages: `type: description`** — enforced by commit-msg hook
16. **Test every handler: happy path + validation-fail + not-found** — minimum 3 cases
17. **Feature folders: `schema → queries → mutations → handlers → api → hooks → index → components`** — no exceptions
18. **Cross-feature imports go through `index.ts`** — never deep-import into another feature
19. **Component filenames are kebab-case** — `home-navbar.tsx`, not `HomeNavbar.tsx`
20. **No hardcoded color values** — semantic tokens only
21. **No hardcoded `px` values** — use Tailwind spacing scale
22. **Use shadcn components, never duplicate them** — extend via `cva` variants
23. **Rate limit every unauthenticated public route by IP** — applied in middleware for `/api/v1/*`; webhooks exempt
24. **`page.tsx` files never use `"use client"`** — kills SSR/SEO; extract interactivity into a child component
25. **Static assets live in `public/`, referenced with absolute paths** — no secrets, no files >1MB
26. **Use `next/image` for raster images and `next/font` for fonts**
27. **Every form uses `react-hook-form` + `zodResolver`** — same Zod schema the route handler validates with; submit via `useMutation`, not raw `fetch`
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

Next.js 16 app with **strict REST APIs and zero server actions** (except the NextAuth sign-in/sign-out forms). Bun = package manager, Node.js = runtime.

| Surface | Where |
|---|---|
| Authenticated UI | `src/app/(platform)/` |
| Public pages (login, terms, privacy) | `src/app/(auth)/`, `src/app/terms`, etc. |
| **Versioned REST API** | `src/app/api/v1/<resource>/route.ts` |
| Webhooks | `src/app/api/webhooks/<vendor>/route.ts` |
| Business logic | `src/features/<name>/handlers.ts` |
| DB reads | `src/features/<name>/queries.ts` |
| DB writes | `src/features/<name>/mutations.ts` |
| Shared utilities | `src/lib/` |
| DB schema | `src/db/schema/` |

---

## Canonical feature reference

Every feature follows:
```
src/features/<name>/
  schema.ts       ← Zod request + response schemas
  queries.ts      ← DB reads
  mutations.ts    ← DB writes
  handlers.ts     ← (ctx, input) → output; pure business logic
  api.ts          ← Client-side fetch wrappers
  hooks/          ← React Query hooks
  index.ts        ← public barrel
  components/
```

**The handler/route split is the core architectural decision:**
- `handlers.ts` is HTTP-agnostic — easy to unit-test, reusable across routes
- `route.ts` is the HTTP adapter — parses + auths + validates + calls handler

Cross-feature imports go through `index.ts`. Never deep-import.

---

## How to... (playbooks)

### Add a new feature
```bash
bun run create-feature <kebab-name>
```
Then follow the printed steps: define the DB table, extend `AuditAction`, build the route handlers under `/api/v1/<name>/`, build components.

### Add a new API endpoint
1. Define request + response schemas in `feature/schema.ts`
2. Write the handler in `feature/handlers.ts` — accepts `(ctx, input)`, returns data or throws `ApiError`
3. Create `src/app/api/v1/<resource>/route.ts` — the HTTP adapter
4. Add the typed fetch wrapper in `feature/api.ts`
5. Add a React Query hook in `feature/hooks/`
6. Test: handler tests cover business logic, hook tests cover client behavior

### Add a server action
**Don't.** This scaffold has exactly one server action surface — `features/auth/actions.ts` for sign-in/sign-out. Every other mutation goes through `/api/v1/*`. If you find yourself wanting a server action, you want a route handler instead.

### Add a DB column or table
1. Edit `src/db/schema/<table>.ts`
2. Run `bunx drizzle-kit generate` — commit both the SQL and `drizzle/meta/`
3. `bunx drizzle-kit migrate` to apply locally

⚠️ Adding `NOT NULL` to a populated table without a default fails on deploy. Make it nullable, backfill, then tighten.

### Add an env var
1. Read it via a lazy memoized getter — never top-level `throw` on missing env
2. Add to `.env.example`
3. Add to the `REQUIRED` list in `src/instrumentation.ts`
4. Add to CI workflow's `env:` block

### Add a webhook handler
1. Route at `src/app/api/webhooks/<vendor>/route.ts` with `export const runtime = "nodejs"`
2. Verify signature against the **raw body** before parsing JSON
3. Make the handler idempotent (webhooks retry)
4. Skip rate limiting for webhook routes — signature verification is the gate

---

## Gotchas (non-obvious)

**`page.tsx` files never use `"use client"`.** Kills SSR/SEO. Extract interactivity into a child component.

**Server components import from `queries.ts` directly, not via the API.** No HTTP roundtrip for SSR. Client components fetch via React Query hooks.

**`handlers.ts` files cannot import `next/headers` or `Request`.** Stay HTTP-agnostic so they're testable without HTTP. Auth context flows in via `ApiContext`.

**API versioning is one-way.** Don't edit `/api/v1/*` once it's published. New breaking changes get a new version directory.

**Response envelope is consistent:** success → `{ data: ... }`, error → `{ error: { code, message } }`. The `apiOk` and `apiError` helpers enforce this — never bypass them.

---

## Commands

| Command | What |
|---|---|
| `bun install` | Install deps |
| `bun run dev` | Start dev server on :3000 |
| `bun run build` | Production build |
| `bun run create-feature <name>` | Scaffold a new feature |
| `bunx vitest run` | Run tests |
| `bunx tsc --noEmit` | Typecheck |
| `bunx next lint` | ESLint |
| `bunx drizzle-kit generate` | Generate migration |
| `bunx drizzle-kit migrate` | Apply migrations |
| `bun audit --audit-level=high` | Security audit |
```

---

### 20. AGENTS.md

Create `AGENTS.md`:

```markdown
# AGENTS.md

Entry point for AI code-review agents.

Read these in order:
1. [`CLAUDE.md`](./CLAUDE.md) — repo orientation
2. [`STANDARDS.md`](./STANDARDS.md) — engineering rules
3. [`REVIEW.md`](./REVIEW.md) — full review rubric

---

## Review guidelines

### 🔴 Block-merge (always raise)

- **`"use server"` outside `features/auth/actions.ts`** — this scaffold is REST-only; every other mutation goes through `/api/v1/*`
- **Auth bypass** — route handlers that skip `requireAuth()`; middleware bypass-list growth without a self-auth justification
- **Webhook signature missing** — Stripe / Slack / GitHub webhooks must verify signatures **before** dispatching
- **Plaintext secret storage** — any new secret column without SHA-256 hashing + `timingSafeEqual` comparison
- **Module-load throws for runtime env vars** — breaks `next build`'s page-data collection
- **`NOT NULL` added to a populated table without a default** — fails on deploy
- **FK violations** — passing non-user UUIDs to `audit_logs.user_id`
- **Hardcoded secrets in code or committed env files**
- **Response NOT using `apiOk` / `apiError`** — bypasses the envelope, breaks every client
- **API route handler missing the try/catch + `apiError(err)` wrapper** — unhandled exceptions leak stack traces
- **`handlers.ts` importing `Request`, `NextResponse`, or `next/headers`** — handlers must stay HTTP-agnostic
- **API route response without Zod-typed return shape** — type drift between client expectations and server output
- **`"use client"` on a `page.tsx` or `layout.tsx`** — kills SSR/SEO

### 🟡 Discuss / suggest

- Missing `logAudit` on a state-changing handler (reads don't need audit; mutations do)
- New `AuditAction` value used without adding it to the union in `src/lib/audit.ts`
- Hardcoded color values (`text-red-500`, `bg-[#1a1a1a]`) — use semantic tokens
- Hardcoded `px` values (`p-[14px]`, `w-[320px]`) — use Tailwind scale
- Component filename in PascalCase — should be kebab-case
- Top-level barrel imports in client/edge code — use sub-path imports
- `useState` + `useEffect` fetching data — should be React Query
- Hand-rolled `useState` form state — every form must use `react-hook-form` + `zodResolver`
- Mutation hook missing `onSuccess: invalidateQueries(...)` — leaves stale cache
- Route handler returning the wrong HTTP status (e.g. 200 instead of 201 for POST that creates)

### ⚪ Skip / don't comment

- Style nits auto-fixed by Prettier
- Theoretical race conditions without a concrete attack path
- Missing tests for trivial helpers
- DoS / resource exhaustion concerns under realistic load
- Outdated transitive dependencies (handled separately)

---

## Files to skip entirely

| Path | Reason |
|---|---|
| `**/drizzle/0*.sql`, `**/drizzle/meta/**` | Generated by `drizzle-kit` |
| `bun.lock`, `package-lock.json`, `yarn.lock` | Lockfiles |
| `**/*.test.ts`, `**/*.test.tsx` | Reviewed by humans |
| `**/_template/**` | Scaffolding source |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Context, not code |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |
| `**/public/images/**` | Static assets |

---

## Confidence calibration

- Read the file, not just the diff
- Don't suggest renames or refactors that aren't behavior changes
- Don't recommend a component that doesn't exist
- Match existing patterns
- If unsure, don't flag unless 🔴 or 🟡
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

- **`"use server"` outside `features/auth/actions.ts`** — REST-only architecture; mutations belong in `/api/v1/*`
- **Auth bypass** in route handlers — missing `requireAuth()`
- **Webhook signature missing** — Stripe / Slack / GitHub
- **Module-load throws for env vars** — breaks `next build`
- **`NOT NULL` on a populated table without default** — fails on deploy
- **FK violations** — non-user UUIDs in `audit_logs.user_id`
- **`apiOk` / `apiError` bypassed** — response envelope broken
- **Missing try/catch + `apiError(err)` in route handlers** — unhandled exceptions leak stacks
- **`handlers.ts` importing HTTP types** (`Request`, `NextResponse`, `next/headers`) — handlers must stay pure
- **`"use client"` on a `page.tsx` / `layout.tsx`** — kills SSR/SEO

### 🟡 Discuss

- Missing `revalidateQueries` on mutation success — leaves stale React Query cache
- Missing `logAudit` on a state-changing handler
- New `AuditAction` not added to the union
- Wrong HTTP status code (e.g. 200 for POST that created — should be 201)
- Hardcoded color/px values — use tokens / Tailwind scale
- Component filename in PascalCase — should be kebab-case
- `useState` + `useEffect` data fetching — should be React Query
- Form without `react-hook-form` + `zodResolver`
- Native `confirm()` / `alert()` — use shadcn `Dialog`
- Duplicate shadcn components — extend via `cva`

### ⚪ Skip

- Auto-fixable style nits
- Theoretical race conditions
- Missing tests for trivial helpers
- DoS / resource exhaustion concerns
- Outdated transitive deps

### Files to skip entirely

| Path | Reason |
|---|---|
| `**/drizzle/0*.sql`, `**/drizzle/meta/**` | Generated |
| `bun.lock`, `package-lock.json` | Lockfiles |
| `**/*.test.ts(x)` | Reviewed by humans |
| `**/_template/**` | Scaffolding source |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Context |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |

---

## Migration / schema PR rubric

### 🔴 Block-merge

- **Schema and SQL drift** — `src/db/schema/<table>.ts` doesn't match the generated SQL
- **`NOT NULL` added without default** on a populated table
- **FK to a table that doesn't exist yet** in the migration order
- **Dropping a column / table referenced by live code** — grep for it
- **Renaming a column without a data-preserving plan**

### 🟡 Discuss

- Enum value added to one side but not the other (TS literal type vs Postgres enum)
- Index on a column without a query that needs it
- Unexpected statements in the generated migration

---

## API contract PR rubric

When a PR touches `/api/v1/`:

### 🔴 Block-merge

- **Changing the response shape of a published endpoint** — that's a v2, not a v1 patch
- **Changing the request body shape required by a published endpoint** — same; goes in v2
- **Removing an endpoint** — deprecate first, remove later
- **Wrong status code** — POST that creates must return 201, not 200; DELETE returns 204

### 🟡 Discuss

- Adding an optional field to a response — fine, but document
- Adding a new endpoint under an existing resource — fine
- Loosening request validation — usually fine but worth a sanity check

---

## Confidence calibration

- Read the file, not just the diff
- Don't recommend renames or refactors that aren't behavior changes
- Don't recommend components that don't exist
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
- [ ] If touching `/api/v1/*`: response shape is unchanged for existing endpoints

## Test plan

<!-- How to verify this works -->

-
```

---

### 23. `next.config.ts`

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  serverExternalPackages: ["pg", "pino"],
  transpilePackages: [],
  images: {
    remotePatterns: [
      { protocol: "https", hostname: "lh3.googleusercontent.com" },
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

  const REQUIRED = [
    "DATABASE_URL",
    "AUTH_SECRET",
    "AUTH_GOOGLE_ID",
    "AUTH_GOOGLE_SECRET",
    "NEXT_PUBLIC_APP_URL",
    "UPSTASH_REDIS_REST_URL",
    "UPSTASH_REDIS_REST_TOKEN",
    "GMAIL_USER",
    "GMAIL_APP_PASSWORD",
  ];

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

  const xff = request.headers["x-forwarded-for"];
  const ip = xff?.split(",")[0]?.trim() ?? request.headers["x-real-ip"] ?? null;

  log.error(
    {
      err: error, digest,
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
  error, reset,
}: { error: Error & { digest?: string }; reset: () => void }) {
  useEffect(() => reportClientError(error, { digest: error.digest }), [error]);

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
      <Button asChild variant="outline"><Link href="/">Go home</Link></Button>
    </div>
  );
}
```

**`src/app/(platform)/loading.tsx`:**
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

### 26. Rate limiting

IP-based sliding window using Upstash Redis. Applied in middleware for the whole `/api/v1/*` surface; per-route tighter limits can be added on top.

**Upstash setup (2 minutes):**
1. Go to [console.upstash.com](https://console.upstash.com) → Create Database
2. Pick a region close to your app
3. Plan: **Free** (10,000 req/day) for dev; **Pay as you go** for prod
4. Go to **REST API** tab → copy both values into `.env.local`

**`src/lib/rate-limit.ts`:**
```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";

const redis = Redis.fromEnv();

export type RateLimitResult =
  | { allowed: true }
  | { allowed: false; retryAfter: number };

export function createRateLimiter(config: { windowMs: number; max: number }) {
  const limiter = new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(config.max, `${config.windowMs}ms`),
    prefix: `rl:${config.windowMs}:${config.max}`,
  });

  return async function check(ip: string): Promise<RateLimitResult> {
    const { success, reset } = await limiter.limit(ip);
    if (!success) {
      return { allowed: false, retryAfter: Math.ceil((reset - Date.now()) / 1000) };
    }
    return { allowed: true };
  };
}
```

**Usage — tighter per-route limit on top of the middleware's broad one:**
```typescript
// src/app/api/v1/contact/route.ts
import type { NextRequest } from "next/server";
import { apiOk, apiError } from "@/lib/api/response";
import { ApiError } from "@/lib/api/errors";
import { createRateLimiter } from "@/lib/rate-limit";
import { getIp } from "@/lib/get-ip";

const contactLimiter = createRateLimiter({ windowMs: 15 * 60 * 1000, max: 5 });

export async function POST(req: NextRequest) {
  try {
    const rl = await contactLimiter(getIp(req));
    if (!rl.allowed) {
      throw new ApiError(429, "rate_limited", "Too many requests. Please try again later.");
    }
    // ... rest of handler
    return apiOk({});
  } catch (err) {
    return apiError(err);
  }
}
```

**Sensible defaults:**

| Route | Window | Max |
|---|---|---|
| Auth (login, signup) | 15 min | 10 |
| Contact / waitlist form | 15 min | 5 |
| Public API (broad limit via middleware) | 1 min | 60 |
| Password reset | 1 hour | 3 |

**Rules:**
- Always use IP as the key — never user ID alone
- Apply to every public route — webhooks are exempt (signature verification is their gate)
- Always return `Retry-After` header — clients respect it
- `"unknown"` IPs share one bucket — intentional safe fallback

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
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - uses: actions/cache@v4
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
      - uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx prettier --check "**/*.{ts,tsx,js,jsx,json,md,css}"

  typecheck:
    name: Typecheck
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - uses: actions/cache@v4
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
      - uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bun audit --audit-level=high

  test:
    name: Test
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx vitest run
        env:
          DATABASE_URL: postgresql://dummy:dummy@localhost/dummy
          AUTH_SECRET: test-secret-at-least-32-chars-long!!
          NEXT_PUBLIC_APP_URL: https://app.test

  build:
    name: Build
    runs-on: ubuntu-latest
    timeout-minutes: 30
    needs: [lint, format, typecheck, audit, test]
    steps:
      - uses: actions/checkout@v4
      - uses: oven-sh/setup-bun@v2
      - uses: actions/cache@v4
        with:
          path: ~/.bun/install/cache
          key: bun-${{ runner.os }}-${{ hashFiles('**/bun.lock') }}
          restore-keys: bun-${{ runner.os }}-
      - run: bun install --frozen-lockfile
      - run: bunx next build
        env:
          DATABASE_URL: postgresql://dummy:dummy@localhost/dummy
          AUTH_SECRET: test-secret-at-least-32-chars-long!!
          NEXT_PUBLIC_APP_URL: https://app.test
          UPSTASH_REDIS_REST_URL: https://dummy.upstash.io
          UPSTASH_REDIS_REST_TOKEN: dummy
```

---

### 28. Email — Nodemailer + Gmail

Transactional email via Gmail SMTP. Free, no third-party signup, works for low-volume apps (password resets, verification emails, notifications).

**Gmail App Password setup:**
1. Enable 2-Step Verification at [myaccount.google.com/security](https://myaccount.google.com/security)
2. Generate an App Password at [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. Copy the 16-char password into `.env.local` as `GMAIL_APP_PASSWORD`

**`src/lib/email.ts`:**
```typescript
import nodemailer, { type Transporter } from "nodemailer";
import { log } from "@/lib/log";

let cachedTransporter: Transporter | null = null;

function getTransporter(): Transporter {
  if (cachedTransporter) return cachedTransporter;
  const user = process.env.GMAIL_USER;
  const pass = process.env.GMAIL_APP_PASSWORD;
  if (!user || !pass) throw new Error("GMAIL_USER and GMAIL_APP_PASSWORD are required");
  cachedTransporter = nodemailer.createTransport({ service: "gmail", auth: { user, pass } });
  return cachedTransporter;
}

export type SendEmailResult =
  | { ok: true; messageId: string }
  | { ok: false; error: string };

export async function sendEmail(params: {
  to: string | string[];
  subject: string;
  html: string;
  text?: string;
  replyTo?: string;
}): Promise<SendEmailResult> {
  const fromName = process.env.EMAIL_FROM_NAME ?? "{{APP_NAME}}";
  const fromAddress = process.env.GMAIL_USER ?? "";
  try {
    const info = await getTransporter().sendMail({
      from: `"${fromName}" <${fromAddress}>`,
      to: params.to,
      subject: params.subject,
      html: params.html,
      text: params.text,
      replyTo: params.replyTo,
    });
    log.info({ to: params.to, subject: params.subject, messageId: info.messageId }, "email.sent");
    return { ok: true, messageId: info.messageId };
  } catch (err) {
    log.error({ to: params.to, subject: params.subject, err }, "email.send_failed");
    return { ok: false, error: err instanceof Error ? err.message : "Unknown error" };
  }
}
```

**Usage from a handler:**
```typescript
import { sendEmail } from "@/lib/email";

export async function sendWelcome(ctx: ApiContext, email: string) {
  const result = await sendEmail({
    to: email,
    subject: "Welcome",
    html: `<h1>Hi there</h1>`,
    text: "Hi there.",
  });
  if (!result.ok) {
    // log + continue; don't fail the user-facing flow on email error
  }
}
```

**Rules:**
- Always send `text` alongside `html` — better deliverability
- Never `await sendEmail` in the critical path — fire-and-forget for non-blocking emails
- Use `replyTo` for support emails
- Gmail SMTP limit: 500 messages/day free, 2000/day Workspace — past that, switch to Resend / Postmark
- Never put secrets in email body — send a link to a server-rendered page instead

---

### 29. SEO + assets

#### Root-layout metadata

**`src/app/layout.tsx`:**
```tsx
import type { Metadata } from "next";
import { Providers } from "@/components/providers";
import "./globals.css";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export const metadata: Metadata = {
  metadataBase: new URL(APP_URL),
  title: { default: "{{APP_NAME}}", template: "%s | {{APP_NAME}}" },
  description: "Short, accurate description of what {{APP_NAME}} does.",
  openGraph: {
    title: "{{APP_NAME}}",
    description: "Short, accurate description of what {{APP_NAME}} does.",
    type: "website", url: APP_URL, siteName: "{{APP_NAME}}",
    images: [{ url: "/opengraph-image", width: 1200, height: 630 }],
  },
  twitter: {
    card: "summary_large_image",
    title: "{{APP_NAME}}",
    description: "Short, accurate description of what {{APP_NAME}} does.",
    images: ["/opengraph-image"],
  },
  robots: {
    index: true, follow: true,
    googleBot: { index: true, follow: true, "max-image-preview": "large" },
  },
};

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body><Providers>{children}</Providers></body>
    </html>
  );
}
```

#### Per-page metadata

```tsx
// src/app/things/[id]/page.tsx
import type { Metadata } from "next";
import { findUserThing } from "@/features/things/queries";

export async function generateMetadata({
  params,
}: { params: Promise<{ id: string }> }): Promise<Metadata> {
  const { id } = await params;
  // Server-side direct DB read — no HTTP roundtrip for SSR
  // (Note: this assumes a public-ish thing; for auth-required pages, get session first)
  return {
    title: `Thing ${id}`,
    description: "Detail page",
  };
}
```

#### `sitemap.ts`

**`src/app/sitemap.ts`:**
```typescript
import type { MetadataRoute } from "next";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  return [
    { url: APP_URL, lastModified: new Date(), changeFrequency: "daily", priority: 1.0 },
    { url: `${APP_URL}/login`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.3 },
    { url: `${APP_URL}/terms`, lastModified: new Date(), changeFrequency: "yearly", priority: 0.2 },
    { url: `${APP_URL}/privacy`, lastModified: new Date(), changeFrequency: "yearly", priority: 0.2 },
  ];
}
```

#### `robots.ts`

```typescript
import type { MetadataRoute } from "next";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [{ userAgent: "*", allow: "/", disallow: ["/api/", "/dashboard", "/settings"] }],
    sitemap: `${APP_URL}/sitemap.xml`,
    host: APP_URL,
  };
}
```

#### Favicon + OG conventions

| Filename | Purpose | Size |
|---|---|---|
| `icon.png` | Browser tab favicon | 32×32 or 512×512 |
| `apple-icon.png` | iOS home-screen icon | 180×180 |
| `opengraph-image.png` | Default OG (Facebook, LinkedIn, Slack) | 1200×630 |
| `twitter-image.png` | Twitter card | 1200×600 |

**Default OG = upscaled favicon** on a brand-colored background. Use [realfavicongenerator.net](https://realfavicongenerator.net).

#### Public folder rules

- **All static assets live in `public/`** — reference with absolute paths (`/logo.png` not `./logo.png`)
- **Never put secrets, API keys, or private docs in `public/`** — served unconditionally
- **Never put files >1MB in `public/`** — use a CDN
- **Use `next/font`** instead of self-hosted font files
- **Use `next/image`** for raster images
- **SVG icons** — inline React components for reuse; `public/icons/` for one-off illustrations

---

### 30. Vercel deployment

**One-time setup:**
1. `vercel link` from the repo root
2. Grab `VERCEL_PROJECT_ID` + `VERCEL_ORG_ID` from `.vercel/project.json`
3. Create a Vercel API token at [vercel.com/account/tokens](https://vercel.com/account/tokens)
4. Add three repo secrets in GitHub: `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `VERCEL_PROJECT_ID`

**`vercel.json`** — disable Vercel's auto-git deploy:
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

---

### 31. Dockerfile + .dockerignore

For deploys outside Vercel.

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

ARG DATABASE_URL
ARG AUTH_SECRET
ARG AUTH_GOOGLE_ID
ARG AUTH_GOOGLE_SECRET
ARG NEXT_PUBLIC_APP_URL
ARG UPSTASH_REDIS_REST_URL
ARG UPSTASH_REDIS_REST_TOKEN
ENV DATABASE_URL=${DATABASE_URL}
ENV AUTH_SECRET=${AUTH_SECRET}
ENV AUTH_GOOGLE_ID=${AUTH_GOOGLE_ID}
ENV AUTH_GOOGLE_SECRET=${AUTH_GOOGLE_SECRET}
ENV NEXT_PUBLIC_APP_URL=${NEXT_PUBLIC_APP_URL}
ENV UPSTASH_REDIS_REST_URL=${UPSTASH_REDIS_REST_URL}
ENV UPSTASH_REDIS_REST_TOKEN=${UPSTASH_REDIS_REST_TOKEN}
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
drizzle/meta
.vercel
Thumbs.db
```

---

### 32. .gitignore additions

```
# Env
.env.local
.env.production
.env*.local

# Drizzle
drizzle/meta/

# Build output
.next/
out/
build/

# Vercel
.vercel
```

---

### 33. Final checks

Run in order:
```bash
bun install
bunx drizzle-kit generate
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
| Bun package manager | Fast installs, Vitest integration. Next.js runtime stays Node.js |
| Husky hooks | Format, lint, types, tests, audit on every push |
| Conventional commits | Enforced by commit-msg hook |
| shadcn/ui + tweakcn theme | CSS variables, no hardcoded colors |
| Drizzle + NeonDB | Schema-first migrations, pooled URL for app, direct URL for migrations |
| NextAuth v5 + Google | Sign-in / sign-out via the only `"use server"` file in the codebase |
| **REST API + zero server actions** | Every mutation goes through `/api/v1/*` |
| **`handlers.ts` / `route.ts` split** | Pure business logic + thin HTTP adapter |
| **Response envelope** | `{ data: ... }` / `{ error: { code, message } }` everywhere via `apiOk` / `apiError` |
| **API versioning** | `/api/v1/`, never edit a published version |
| **HTTP status conventions** | 201 for POST creates, 204 for DELETE, 404 for scoped-out, etc. |
| `apiFetch` client | Typed fetch wrapper with Zod response validation |
| Zod conventions | One schema, two consumers — `safeParse` on server, `zodResolver` on client |
| Forms | `react-hook-form` + `zodResolver` + `useMutation` — no inline `useState` |
| React Query | Server state with cache, no `useState`/`useEffect` fetching |
| Feature template | `schema → queries → mutations → handlers → api → hooks → index → components` |
| `bun run create-feature` | Scaffolder copies template, replaces tokens, prints next steps |
| Pino logger | Structured JSON, GCP-severity compatible, secret redaction |
| Client error reporting | `/api/log/client-error` + `reportClientError()` |
| Audit trail | Every state-changing handler calls `logAudit` |
| Upstash Redis | Serverless Redis for rate limiting — IP-based, applied in middleware |
| Vitest + mock builders | Handler tests, hook tests, schema tests — 3-case minimum |
| Styling conventions | Kebab-case files, no hardcoded px/colors, mobile-first |
| `next.config.ts` | `standalone` output, `serverExternalPackages`, security headers |
| `instrumentation.ts` | Startup env validation + global `onRequestError` capture |
| Error/loading pages | `error.tsx`, `global-error.tsx`, `not-found.tsx`, `loading.tsx` |
| SEO | `sitemap.ts`, `robots.ts`, OG image conventions, per-page metadata |
| Governance docs | `CLAUDE.md`, `STANDARDS.md`, `AGENTS.md`, `REVIEW.md`, PR template |
| Email | Nodemailer + Gmail SMTP via App Password |
| Vercel deploy | GitHub Actions workflow + `vercel.json` disabling auto-deploy |
| Dockerfile | Multi-stage Bun→Node build for non-Vercel hosts |
| CI workflow | Parallel lint/format/typecheck/audit/test jobs |

## PROMPT END
