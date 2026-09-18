# Next.js Server-Actions Scaffold Prompt

Copy-paste this entire prompt to scaffold a production-ready Next.js app with **server actions** (no public REST API).
Replace `{{APP_NAME}}` with your app name (kebab-case, e.g. `my-app`).

**Use this when:** the app is Next.js-only with no plan for mobile / third-party HTTP consumers. Mutations are server actions (`ActionResult`). Server components read via `queries.ts`. Webhooks + health + client-error are the only extra HTTP routes.

**Use the REST `/api/v1` prompt instead when:** a mobile client, third-party integrator, or non-TS client must call the same API.

**Use the frontend-only prompt instead when:** you have no first-party database or auth.

---

## PROMPT START

Scaffold a production-ready Next.js app called `{{APP_NAME}}` using **server actions** for every mutation. Follow every instruction exactly — don't add extras, don't skip steps.

Non-interactive CLIs only: `bunx shadcn@latest init -d`. Do not open tweakcn / Neon / Google / Upstash / Vercel in a browser — stub `.env.example` and leave a post-scaffold checklist. Ship a default `globals.css` token set (shadcn Slate + CSS variables).

**Write `CLAUDE.md`, `AGENTS.md`, `STANDARDS.md`, and `REVIEW.md` in full from the templates in this prompt.** Do not stub. They must match §0. Do **not** add Sentry, PostHog, or a Redis response cache.

---

### 0. Exception handling, errors, logging, observability (read first)

Four layers. Do not mix them.

```
return fail() / throw Error   →   withAction catch
        │                              ├─ fail(...)           → { success: false, error } (user-safe)
        │                              └─ unknown Error       → log.error + fail("Something went wrong.")
log.info / warn / error       →   stdout JSON. Operators only.
logAudit(...)                 →   audit_logs table. Durable "who did what".
```

**Exceptions**

- Expected failure returns `fail("…")` — validation, auth, not-found, unique conflict. That is not a crash.
- Unexpected failure is a real `Error`. `withAction` is the **only** `try/catch` around an action. Actions do not catch-and-swallow. Never throw `err.message` across the server/client boundary.
- Unique-key collisions (`pg` `23505`) → `fail("Already exists.")`.
- Process: `uncaughtException` → `log.fatal` in Node-only `process-handlers.ts` (not inside `instrumentation.ts`). `unhandledRejection` → `log.error`.

**Error handling (what the client sees)**

```ts
{ success: true, data }
{ success: false, error: "Thing not found." }   // canned English, never err.message
```

Other users' ids → not-found fail, not a leaky "forbidden". Forms show `result.error` via `form.setError` + `toast.error`.

**Logging**

Pino only. `console.*` is an ESLint error. `log.<level>({ ...context }, "feature.verb")`.

| Level | When |
|---|---|
| `fatal` | Process is going down |
| `error` | This action/request actually failed |
| `warn` | Recoverable (expected fail() already returned, Redis down) |
| `info` | State change (`things.created`) |
| `debug` | Off in prod unless `LOG_LEVEL=debug` |

Writes: `log.info` **and** `logAudit`. Reads do neither. IP belongs on `audit_logs` and on `client.error_reported` / `request.error` — not on every `log.info`.

**Observability**

- Dual fields: `level` + GCP `severity`. `service` from `DD_SERVICE`.
- `APP_ENV=local|dev|prod` is a **build ARG**. Browser source maps on local/dev, **off prod**. Not `NODE_ENV`.
- `/api/health` → `SELECT 1` → 503 if DB down.
- Client errors POST same-origin to `/api/log/client-error`.
- Next `onRequestError` logs as `request.error`.
- Redis is **rate limiting only**. No Redis/CDN cache of authenticated pages/JSON. Cache for the UI is `revalidatePath` after writes.
- Do not add Sentry or PostHog until a DSN/key exists.

---

### 1. Init

> **Bun is the package manager and script runner — Next.js still runs on Node.js.**
> `bun install`, `bun add`, `bunx` are all Bun. But `next dev` / `next build` / `next start` execute under Node.js internally. Never use `bun run --bun next dev` expecting a Bun runtime for Next.js — it's unsupported and causes hard-to-debug issues. Bun's value here is fast installs, fast `bunx`, and Vitest integration. The app runtime is Node.js.

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
  "@hookform/resolvers": "^5"
}
```

**Dev deps:**
```json
{
  "drizzle-kit": "^0.31",
  "vitest": "^3",
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
**/.turbo/
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
2. Pick a region close to your app's hosting region (latency matters)
3. Copy the connection string from **Connection Details → Connection string**
4. Use the **pooled** connection string (ends in `-pooler.neon.tech`) for the app. Use the direct string only for `drizzle-kit migrate`.

Add to `.env.example`:
```bash
# Pooled — used by the app at runtime (handles serverless connection limits)
DATABASE_URL=postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/dbname?sslmode=require

# Direct — used only by drizzle-kit for migrations (bypasses pooler)
DATABASE_URL_DIRECT=postgresql://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require
```

**Switching providers later** — replace both `DATABASE_URL` values. Nothing else changes. Neon-specific notes:
- The `?sslmode=require` at the end is Neon's requirement — other providers may not need it
- Neon's free tier pauses the database after 5 minutes of inactivity — the first query after a pause takes ~1s to wake it. Upgrade to a paid plan for production.

Create `src/db/index.ts` — **lazy**. `next build` imports this module; a top-level throw on `DATABASE_URL` kills CI.
```typescript
import { Pool } from "pg";
import { drizzle } from "drizzle-orm/node-postgres";
import * as schema from "./schema";
import { databaseUrl } from "@/lib/env";

type Db = ReturnType<typeof drizzle<typeof schema>>;

let pool: Pool | undefined;
let cached: Db | undefined;

function sslFor(url: string) {
  const local = url.includes("localhost") || url.includes("127.0.0.1");
  if (local || url.includes("sslmode=disable")) return false;
  return { rejectUnauthorized: true } as const;
}

/** Call from actions / queries — never at module import. */
export function getDb(): Db {
  if (cached) return cached;
  const url = databaseUrl();
  pool = new Pool({ connectionString: url, ssl: sslFor(url) });
  cached = drizzle(pool, { schema });
  return cached;
}

export * from "./schema";
```

Use `getDb().query` / `getDb().insert` everywhere the old code said `db.` (queries, actions, audit, health).

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

// Use the direct (non-pooled) URL for migrations — drizzle-kit needs a
// persistent connection that the pooler doesn't support. Falls back to
// DATABASE_URL for local dev where there's only one URL.
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

**Google Cloud Console setup (do this first):**
1. Go to [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID → Application type: **Web application**
3. Add Authorized redirect URIs:
   - `http://localhost:3000/api/auth/callback/google` (dev)
   - `https://your-domain.com/api/auth/callback/google` (prod)
4. Copy **Client ID** → `AUTH_GOOGLE_ID`
5. Copy **Client Secret** → `AUTH_GOOGLE_SECRET`
6. Go to **OAuth consent screen** → add your app name, logo, and support email. The default `email` and `profile` scopes are what NextAuth requests.

**TypeScript type augmentation** — create `src/types/next-auth.d.ts`:
```typescript
import type { DefaultSession } from "next-auth";

declare module "next-auth" {
  interface Session {
    user: {
      id: string;
    } & DefaultSession["user"];
  }
}

declare module "next-auth/jwt" {
  interface JWT {
    id: string;
  }
}
```

**`src/features/auth/lib/config.ts`** — edge-safe config (no DB imports):
```typescript
import type { NextAuthConfig } from "next-auth";
import Google from "next-auth/providers/google";

export const authConfig: NextAuthConfig = {
  providers: [
    Google({
      clientId: process.env.AUTH_GOOGLE_ID,
      clientSecret: process.env.AUTH_GOOGLE_SECRET,
      authorization: {
        params: {
          // Force account picker every time — prevents silent re-auth
          // with a stale Google session when the user wants to switch accounts.
          prompt: "select_account",
        },
      },
    }),
  ],
  session: { strategy: "jwt" }, // JWT only. Adapter persists users/accounts, not DB sessions.
  // Required for deployments behind a reverse proxy (Cloud Run, Railway, Fly,
  // Vercel) — the host header is set by the load balancer, not the container.
  trustHost: true,
  callbacks: {
    async jwt({ token, user }) {
      // `user` is only present on the first sign-in — persist the DB user id
      // into the JWT so subsequent requests don't need a DB lookup.
      if (user?.id) token.id = user.id;
      return token;
    },
    async session({ session, token }) {
      if (token.id) session.user.id = token.id;
      return session;
    },
  },
  pages: {
    signIn: "/login",
    error: "/login", // NextAuth appends ?error=... — handled in the login page
  },
};
```

**`src/features/auth/index.ts`** — adapter via lazy `getDb()` so import does not throw at build:
```typescript
import NextAuth from "next-auth";
import { DrizzleAdapter } from "@auth/drizzle-adapter";
import { getDb } from "@/db";
import { authConfig } from "./lib/config";

const dbProxy = new Proxy({} as ReturnType<typeof getDb>, {
  get(_target, prop) {
    return Reflect.get(getDb() as object, prop);
  },
});

export const { handlers, auth, signIn, signOut } = NextAuth({
  ...authConfig,
  adapter: DrizzleAdapter(dbProxy as never),
});
```

**`src/features/auth/actions.ts`** — auth server actions:
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

**`src/app/api/auth/[...nextauth]/route.ts`:**
```typescript
import { handlers } from "@/features/auth";
export const { GET, POST } = handlers;
```

**`src/app/(auth)/login/page.tsx`** — plain login page, no hardcoded colours:
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

        <p className="text-center text-xs text-muted-foreground">
          By continuing you agree to our{" "}
          <a href="/terms" className="underline underline-offset-4 hover:text-foreground">
            Terms
          </a>{" "}
          and{" "}
          <a href="/privacy" className="underline underline-offset-4 hover:text-foreground">
            Privacy Policy
          </a>
          .
        </p>
      </div>
    </div>
  );
}
```

**Sign-out button** — import `signOutUser` from auth actions:
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

**Reading the session in server components:**
```typescript
import { auth } from "@/features/auth";

export default async function Page() {
  const session = await auth();
  // session.user.id, session.user.name, session.user.email, session.user.image
}
```

Create `src/middleware.ts` — combined auth guard + IP rate limiting:
```typescript
import NextAuth from "next-auth";
import { NextResponse, type NextRequest } from "next/server";
import { authConfig } from "@/features/auth/lib/config";
import { createRateLimiter } from "@/lib/rate-limit";
import { getIp } from "@/lib/get-ip";

const { auth } = NextAuth(authConfig);

// Broad rate limit applied to all public API routes before auth runs.
// Tighter per-route limiters can be added on top of this in individual
// route handlers for sensitive endpoints (auth, forms, password reset).
const apiLimiter = createRateLimiter({ windowMs: 60_000, max: 60 });

export default auth(async (req) => {
  const { pathname } = req.nextUrl;
  const isAuthRoute = pathname.startsWith("/api/auth");
  const isWebhook = pathname.startsWith("/api/webhooks");
  const isHealth = pathname.startsWith("/api/health");
  const isClientError = pathname.startsWith("/api/log/client-error");
  const isPublicPage = ["/login", "/terms", "/privacy"].includes(pathname);

  const isPublicApi =
    pathname.startsWith("/api/") && !isAuthRoute && !isWebhook && !isHealth;
  if (isPublicApi) {
    const result = await apiLimiter(getIp(req));
    if (!result.allowed) {
      return NextResponse.json(
        { error: "Too many requests." },
        { status: 429, headers: { "Retry-After": String(result.retryAfter) } }
      );
    }
  }

  if (isAuthRoute || isWebhook || isHealth || isClientError || isPublicPage) return;
  const isLoggedIn = !!req.auth;
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

**`src/lib/action-result.ts`:**
```typescript
export type ActionResult<T = void> = { success: true; data: T } | { success: false; error: string };

export const fail = (error: string): { success: false; error: string } => ({
  success: false,
  error,
});

export const ok = <T>(data: T): { success: true; data: T } => ({
  success: true,
  data,
});

export const firstError = (issues: { message: string }[], fallback: string): string =>
  issues[0]?.message ?? fallback;

export function isPgUniqueViolation(err: unknown): boolean {
  return typeof err === "object" && err !== null && "code" in err && err.code === "23505";
}
```

**`src/lib/with-action.ts`** — the only `try/catch` around a server action.
```typescript
import { log } from "@/lib/log";
import { fail, type ActionResult } from "@/lib/action-result";

export async function withAction<T>(
  name: string,
  fn: () => Promise<ActionResult<T>>
): Promise<ActionResult<T>> {
  try {
    return await fn();
  } catch (err) {
    log.error({ err }, `${name}.unhandled_error`);
    return fail("Something went wrong.");
  }
}
```

**`src/lib/env.ts`** — lazy getters. Call from a request/action, never at module import.
```typescript
function required(name: string): () => string {
  let cached: string | undefined;
  return () => {
    if (cached !== undefined) return cached;
    const value = process.env[name];
    if (!value) throw new Error(`${name} is not set`);
    cached = value;
    return value;
  };
}

export const databaseUrl = required("DATABASE_URL");
export const authSecret = required("AUTH_SECRET");

export function webhookSecret(): string | undefined {
  return process.env.WEBHOOK_SECRET;
}
```

**`src/lib/app-environment.ts`** — `NODE_ENV` cannot tell Cloud Run `dev` from `prod`.
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
      return { service: process.env.DD_SERVICE ?? "{{APP_NAME}}" };
    },
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  redact: {
    paths: [
      "password", "*.password", "*.*.password",
      "token", "*.token", "*.*.token",
      "accessToken", "*.accessToken", "refreshToken", "*.refreshToken",
      "apiKey", "*.apiKey", "api_key", "*.api_key",
      "secret", "*.secret", "webhookSecret", "*.webhookSecret",
      "authorization", "*.authorization", "headers.authorization", "headers.cookie",
      "privateKey", "*.privateKey", "credentials", "*.credentials",
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

export function getClientIp(headers: Headers): string | null {
  const xff = headers.get("x-forwarded-for");
  return xff?.split(",")[0]?.trim() || headers.get("x-real-ip") || null;
}
```

**`src/lib/get-ip.ts`** — used by rate limiter and elsewhere:
```typescript
import type { NextRequest } from "next/server";

// x-forwarded-for is "client, proxy1, proxy2" — first entry is the real IP
// behind GCP/Vercel/Cloudflare load balancers. Falls back to x-real-ip,
// then "unknown" so the rate limiter still fires (all unknowns share a bucket).
export function getIp(req: NextRequest): string {
  return (
    req.headers.get("x-forwarded-for")?.split(",")[0]?.trim() ??
    req.headers.get("x-real-ip") ??
    "unknown"
  );
}
```

**`src/lib/process-handlers.ts`** — Node-only. Do not put `process.on` in `instrumentation.ts`.
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

**`src/lib/audit.ts`:**
```typescript
import { headers } from "next/headers";
import { getDb } from "@/db";
import { auditLogs } from "@/db/schema";
import { getClientIp } from "@/lib/log";

// Add your own AuditAction values here as you build features.
// Format: "<feature>.<verb>" — e.g. "post.created", "user.deleted"
export type AuditAction =
  | "auth.login"
  | "auth.logout"
  | "user.updated"
  | "user.deleted";
  // Extend per feature — e.g. "post.created", "comment.deleted", "billing.subscribed"

type LogAuditParams = {
  // null for system-triggered events (webhooks, crons, background jobs).
  // The audit_logs.user_id column is FK→users.id, so passing a non-user UUID
  // violates the constraint and crashes the caller.
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
  await getDb().insert(auditLogs).values({
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

Zod is the **only** validation library. Use it at every system boundary — user input, webhook bodies, external API responses, env vars. Never validate internal function calls; trust TypeScript there.

**Rules — enforce all of these:**

```typescript
// ✅ Always safeParse — never .parse() which throws
const parsed = CreateThingSchema.safeParse(input);
if (!parsed.success) return fail(firstError(parsed.error.issues, "Invalid input"));

// ✅ Every string field gets .trim() + .min(1) + .max() with a custom user-facing message
const NameSchema = z
  .string({ message: "Name is required" })
  .trim()
  .min(1, { message: "Name is required" })
  .max(120, { message: "Name must be 120 characters or fewer" });

// ✅ Optional nullable text — reusable helper in every schema file
const optionalText = (max: number, label: string) =>
  z
    .string()
    .max(max, { message: `${label} must be ${max.toLocaleString("en-US")} characters or fewer` })
    .nullable()
    .optional();

// ✅ UUIDs always use this — consistent error message across all features
const UuidSchema = z.string().uuid({ message: "Invalid id" });

// ✅ Always export inferred types alongside schemas
export const CreateThingInputSchema = z.object({
  name: NameSchema,
  description: optionalText(2_000, "Description"),
});
export type CreateThingInput = z.infer<typeof CreateThingInputSchema>;

// ❌ Never .parse() — it throws a ZodError which crosses the server/client
//    boundary as an unstructured 500, not a user-facing message
const parsed = CreateThingSchema.parse(input); // WRONG
```

**Error message rules:**
- Messages are **user-facing** — no TypeScript jargon, no "must satisfy schema"
- Required field: `"Name is required"` not `"name must be a string"`
- Length limit: `"Name must be 120 characters or fewer"` not `"String too long"`
- UUID: `"Invalid id"` not `"Invalid uuid"`

**Where Zod runs:**

| Boundary | Use Zod? |
|---|---|
| Server action input | ✅ Always |
| API route body | ✅ Always |
| Webhook body (after signature check) | ✅ Always |
| External API response | ✅ Always |
| Env var validation | ✅ At first use |
| Internal function calls | ❌ Trust TypeScript |
| DB query results via Drizzle | ❌ Drizzle types it |

---

### 10. Logger conventions

Pino is the **only** logger. `console.log`, `console.error`, `console.warn` are banned server-side — ESLint should flag them.

**Always: context object first, message string second**
```typescript
// ✅ Structured context first — every field is queryable in your log backend
log.info({ userId, thingId: row.id }, "things.created");
log.error({ userId, thingId, err }, "things.create_failed");
log.warn({ provider }, "integration.token_expired");

// ❌ String interpolation — kills structured search
log.info(`Created thing ${row.id} for user ${userId}`); // WRONG
log.error("Failed: " + err.message);                    // WRONG
```

**Event name format: `feature.verb` or `feature.noun_verb`**
```typescript
// ✅ Dotted namespace — grep-able, consistent
"auth.login"
"auth.session_invalid"
"things.created"
"things.delete_failed"
"stripe.webhook_received"

// ❌ Plain strings — not queryable
"Created a thing"
"Login failed"
```

**Log level guide:**

| Level | When | Always include |
|---|---|---|
| `log.fatal` | Process is going down (uncaught exception) | `err` |
| `log.error` | Request failed — user/system shouldn't ignore it | `err` + context |
| `log.warn` | Recoverable degradation (token refresh failed, retrying) | context |
| `log.info` | State change worth an audit trail (created, updated, fired) | `userId` |
| `log.debug` | Dev-only detail — off in prod by default | anything |

**Never log sensitive values — even at debug:**
```typescript
// ❌ The redactor strips known paths but won't save you from new field names
log.info({ apiKey: input.apiKey }, "things.key_set");    // WRONG — add to redact list or don't log
log.debug({ body: rawWebhookBody }, "webhook.received"); // WRONG — body may contain secrets

// ✅ Log the shape, not the value
log.info({ userId, provider, keyPrefix: key.slice(0, 4) }, "integration.key_set");
```

**Child logger for request-scoped context** — bind once, use throughout:
```typescript
const reqLog = log.child({ requestId, userId });
reqLog.info("things.created");
reqLog.error({ err }, "things.create_failed");
```

**Client-side errors** — never `console.error`, never swallow:
```typescript
// In client components:
import { toast } from "sonner"; // or whichever toast library shadcn installed

try {
  const result = await createThing(input);
  if (!result.success) toast.error(result.error);
} catch (err) {
  reportClientError(err); // src/lib/report-client-error.ts
  toast.error("Something went wrong. Please try again.");
}
```

Create `src/lib/report-client-error.ts`:
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

Create `src/app/api/log/client-error/route.ts` — same-origin, Zod, size cap, IP rate limit:
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

### 11. Feature folder pattern

Create `src/features/_template/` with these files. This is the canonical shape every new feature copies.

**`schema.ts`** — Zod input validation only. Every string field gets `.max()` with a custom message. Every error message is user-facing.

**`queries.ts`** — Read-only Drizzle queries. Export typed row types.

**`actions.ts`** — Server actions (`"use server"` at the top of the file, never inline). Wrap every exported action in `withAction`. Inside:

1. `auth()` → check `session?.user?.id` → `fail("Unauthorized")`
2. `Schema.safeParse(input)` → `fail(firstError(...))`
3. DB write via `getDb()`. Map `23505` with `isPgUniqueViolation` → `fail("Already exists.")`
4. `log.info({ userId, ...context }, "feature.verb")`
5. `await logAudit({ userId, action: "feature.verb", resource: id })`
6. `revalidatePath(...)` on affected paths
7. `return ok(result)`

No second `try/catch`. Never return `err.message`.

**`index.ts`** — Narrow public barrel: page components + actions + types only. Cross-feature imports go through this file, never deep-imports.

**`components/`** — Client + server components for this feature only.

**Token placeholders inside `_template/`** — files reference these strings so the scaffolder can find-and-replace them. Use them consistently across `schema.ts`, `queries.ts`, `actions.ts`, `index.ts`, and `components/`:

| Token | Replaced with | Example for `billing-events` |
|---|---|---|
| `_template` | feature folder name (kebab) | `billing-events` |
| `Things` | PascalCase plural | `BillingEvents` |
| `Thing` | PascalCase singular | `BillingEvent` |
| `things` | folder name reused as plural | `billing-events` |
| `thing` | kebab-case singular | `billing-event` |

#### Feature scaffolder script

Create `scripts/create-feature.ts`:

```typescript
#!/usr/bin/env bun
//
// Scaffolds a new feature from src/features/_template/.
//
//   bun run create-feature <kebab-name>
//   bun run create-feature billing-events
//   bun run create-feature people --singular Person   (irregular plurals)

import * as fs from "node:fs";
import * as path from "node:path";
import { fileURLToPath } from "node:url";

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const ROOT = path.resolve(__dirname, "..");

// ── Args ───────────────────────────────────────────────────────────────────
const args = process.argv.slice(2);
const flags: Record<string, string | true> = {};
const positionals: string[] = [];

for (let i = 0; i < args.length; i++) {
  const a = args[i];
  if (a.startsWith("--")) {
    const key = a.slice(2);
    const next = args[i + 1];
    if (next && !next.startsWith("--")) {
      flags[key] = next;
      i++;
    } else {
      flags[key] = true;
    }
  } else {
    positionals.push(a);
  }
}

const name = positionals[0];
if (!name || flags.help === true || flags.h === true) {
  printUsage();
  process.exit(name ? 0 : 1);
}

// ── Validate ───────────────────────────────────────────────────────────────
if (!/^[a-z][a-z0-9-]*[a-z0-9]$/.test(name)) {
  fail(
    `Invalid feature name "${name}". Use kebab-case: lowercase letters, ` +
      `digits, and dashes only; must start with a letter and end with a letter or digit.`
  );
}

if (name === "_template") fail(`The name "_template" is reserved.`);

const RESERVED = new Set(["app", "lib", "test", "components", "auth", "db", "types"]);
if (RESERVED.has(name)) fail(`The name "${name}" is reserved.`);

const templateDir = path.join(ROOT, "src/features/_template");
const targetDir = path.join(ROOT, `src/features/${name}`);

if (!fs.existsSync(templateDir)) {
  fail(`Template not found at ${templateDir}. Did you delete src/features/_template?`);
}
if (fs.existsSync(targetDir)) {
  fail(`Feature already exists at ${targetDir}. Pick a new name or delete the existing folder.`);
}

// ── Naming variants ────────────────────────────────────────────────────────
const pascalPlural = name
  .split("-")
  .map((s) => s.charAt(0).toUpperCase() + s.slice(1))
  .join("");

const pascalSingularDefault = pascalPlural.replace(/s$/, "") || pascalPlural;
const pascalSingular = typeof flags.singular === "string" ? flags.singular : pascalSingularDefault;

const kebabSingular = pascalSingular.replace(/([a-z])([A-Z])/g, "$1-$2").toLowerCase();

// Order matters — longer/more specific patterns first so we don't double-
// replace ("things" before "thing").
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

// ── Copy the template tree ─────────────────────────────────────────────────
let filesWritten = 0;

function copyTree(srcDir: string, dstDir: string): void {
  fs.mkdirSync(dstDir, { recursive: true });
  for (const entry of fs.readdirSync(srcDir, { withFileTypes: true })) {
    const srcPath = path.join(srcDir, entry.name);
    const newName = applyReplacements(entry.name);
    const dstPath = path.join(dstDir, newName);
    if (entry.isDirectory()) {
      copyTree(srcPath, dstPath);
    } else {
      const content = fs.readFileSync(srcPath, "utf8");
      fs.writeFileSync(dstPath, applyReplacements(content));
      filesWritten++;
    }
  }
}

copyTree(templateDir, targetDir);

// ── Output ─────────────────────────────────────────────────────────────────
const relTarget = path.relative(ROOT, targetDir);

console.log(`Created ${relTarget}/  (${filesWritten} files)`);
console.log("");
console.log("Naming used:");
console.log(`  feature name:    ${name}`);
console.log(`  Plural noun:     ${pascalPlural}`);
console.log(`  Singular noun:   ${pascalSingular}    (override with --singular <Name>)`);
console.log("");
console.log("Next steps:");
console.log(`  1. Define the DB table in src/db/schema/${name}.ts and re-export from src/db/schema/index.ts`);
console.log(`  2. Add AuditAction values to src/lib/audit.ts (e.g. "${kebabSingular}.created", "${kebabSingular}.deleted")`);
console.log(`  3. Open src/features/${name}/queries.ts, actions.ts, index.ts — uncomment and adapt`);
console.log(`  4. Build out src/features/${name}/components/`);
console.log(`  5. Once the page exports compile, add the app route:`);
console.log(`     src/app/(platform)/${name}/page.tsx →`);
console.log(`       import { ${pascalPlural}ListPage } from "@/features/${name}";`);
console.log(`       export default ${pascalPlural}ListPage;`);
console.log(`  6. bun run db:generate && bun run db:migrate`);
console.log(`  7. bunx tsc --noEmit && bunx vitest run`);

// ── Helpers ────────────────────────────────────────────────────────────────
function fail(message: string): never {
  process.stderr.write(`error: ${message}\n`);
  process.exit(1);
}

function printUsage(): void {
  console.log(
    [
      "Scaffold a new feature from src/features/_template/.",
      "",
      "Usage:",
      "  bun run create-feature <kebab-name> [--singular <PascalNoun>]",
      "",
      "Examples:",
      "  bun run create-feature alerts",
      "  bun run create-feature billing-events",
      "  bun run create-feature people --singular Person",
    ].join("\n")
  );
}
```

Wire it up in `package.json`:
```json
{
  "scripts": {
    "create-feature": "bun scripts/create-feature.ts"
  }
}
```

**Usage:**
```bash
bun run create-feature alerts              # standard kebab name
bun run create-feature billing-events      # multi-word
bun run create-feature people --singular Person   # irregular plural
```

**What the script does:**
1. Validates the name (kebab-case, not reserved, not already taken)
2. Copies `src/features/_template/` → `src/features/<name>/`, replacing all five tokens in both filenames AND file contents
3. Prints the remaining manual steps — DB schema, audit-action values, route wiring, migrations

**What the script deliberately does NOT do:**
- Generate the Drizzle table — you have to design your columns
- Add `AuditAction` enum values — depends on your verbs
- Create the app route — that import fails typecheck until you uncomment the page export in `index.ts`. Add the route as the final step.

#### Forms — react-hook-form + Zod (mandatory pattern)

**Every form uses `react-hook-form` + the same Zod schema the server action uses.** No exceptions. Hand-rolled `useState` form state is banned — it duplicates validation, loses field-level error tracking, and drifts from the server's schema.

The pattern: one Zod schema in `feature/schema.ts` is the source of truth. The **server action** validates with it via `safeParse`. The **client form** validates with it via `zodResolver`. Same rules on both sides — typo a `.max()` once and both ends update.

**Example form using shadcn's `<Form>` primitives:**

```tsx
// src/features/things/components/create-thing-form.tsx
"use client";

import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { toast } from "sonner";
import { CreateThingInputSchema, type CreateThingInput } from "@/features/things/schema";
import { createThing } from "@/features/things/actions";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";

export function CreateThingForm() {
  const form = useForm<CreateThingInput>({
    resolver: zodResolver(CreateThingInputSchema),
    defaultValues: { name: "", description: "" },
  });

  async function onSubmit(values: CreateThingInput) {
    const result = await createThing(values);
    if (!result.success) {
      // Server-side validation or business-rule error — surface it on the form
      form.setError("root", { message: result.error });
      return;
    }
    toast.success("Thing created");
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
              <FormControl>
                <Input {...field} />
              </FormControl>
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
              <FormControl>
                <Textarea {...field} value={field.value ?? ""} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        {form.formState.errors.root && (
          <p className="text-sm text-destructive">{form.formState.errors.root.message}</p>
        )}

        <Button type="submit" disabled={form.formState.isSubmitting}>
          {form.formState.isSubmitting ? "Creating..." : "Create"}
        </Button>
      </form>
    </Form>
  );
}
```

**Rules — enforce all of these:**

- **One schema, two consumers.** The Zod schema lives in `feature/schema.ts`. The action imports it for `safeParse`. The form imports it for `zodResolver`. Never duplicate the rules.
- **Always `zodResolver`.** Never hand-roll `validate` functions or `useState`-driven validation. `useForm({ resolver: zodResolver(Schema) })` is the only way.
- **Default values must satisfy the schema's shape.** Pass `defaultValues` matching every field — `""` for required strings, `null` or `undefined` for optionals. Skipping `defaultValues` triggers "uncontrolled to controlled" React warnings and breaks form reset.
- **Server-side error → `form.setError("root", ...)`**. Field-level Zod errors render automatically via `<FormMessage />`. For business-rule errors that come back from the action (quota, conflict, not-found), surface them on the form's `root` error.
- **Disable submit while pending.** `disabled={form.formState.isSubmitting}` on the submit button. Show a pending label so the user knows it's in flight.
- **Toast on success, reset on success.** Confirms the write landed and clears the form for the next entry.
- **For optional nullable fields**, controlled inputs need `value={field.value ?? ""}` — React doesn't accept `null` as a value prop.

---

### 12. App structure

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
    error.tsx
    global-error.tsx
    not-found.tsx
  features/
    auth/
      lib/config.ts
      actions.ts
      index.ts
    _template/
      schema.ts
      queries.ts
      actions.ts
      index.ts
      components/
  lib/
    action-result.ts
    with-action.ts
    env.ts
    app-environment.ts
    log.ts
    process-handlers.ts
    audit.ts
    get-ip.ts
    rate-limit.ts
    webhook.ts
    report-client-error.ts
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

**`src/app/(platform)/layout.tsx`** — auth-gated shell:
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
import { getDb } from "@/db";
import { sql } from "drizzle-orm";

export const runtime = "nodejs";

export async function GET() {
  try {
    await getDb().execute(sql`SELECT 1`);
    return NextResponse.json({ ok: true });
  } catch {
    return NextResponse.json({ ok: false }, { status: 503 });
  }
}
```

---

### 13. Vitest

Create `vitest.config.ts`:
```typescript
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "node",
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

Create `src/test/mocks/auth.ts`:
```typescript
import type { Session } from "next-auth";

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

Add test script to `package.json`:
```json
{ "test": "vitest run" }
```

Every action test file follows this pattern: `vi.mock` auth + db + next/cache at the top, then 3 cases per action — happy path, auth-fail (`auth` returns null), validation-fail (bad input).

---

### 14. Environment variables

Create `.env.example`:
```bash
# Database (NeonDB)
# Pooled — used by the app at runtime
DATABASE_URL=postgresql://user:pass@ep-xxx-pooler.region.aws.neon.tech/dbname?sslmode=require
# Direct — used only by drizzle-kit for migrations (bypasses pooler)
DATABASE_URL_DIRECT=postgresql://user:pass@ep-xxx.region.aws.neon.tech/dbname?sslmode=require

# Auth
AUTH_SECRET=generate-with-openssl-rand-base64-32
AUTH_GOOGLE_ID=your-google-oauth-client-id
AUTH_GOOGLE_SECRET=your-google-oauth-client-secret

# App
NEXT_PUBLIC_APP_URL=http://localhost:3000
APP_ENV=local
LOG_LEVEL=debug
DD_SERVICE={{APP_NAME}}
# APP_LOG_FILE=/shared-volume/logs/app.log

# Webhook HMAC (raw-body verify). Optional until you add a vendor.
# WEBHOOK_SECRET=

# Upstash Redis (rate limiting only — not a response cache)
UPSTASH_REDIS_REST_URL=https://your-db.upstash.io
UPSTASH_REDIS_REST_TOKEN=your-token

# Email (Gmail SMTP via App Password)
GMAIL_USER=your-app@gmail.com
GMAIL_APP_PASSWORD=xxxxxxxxxxxxxxxx
EMAIL_FROM_NAME={{APP_NAME}}
```

Create `.env.local` from `.env.example` and fill in real values.

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

These rules apply to every `.tsx` file. Enforce them in code review.

---

**Component file naming — always kebab-case:**
```
✅ home-navbar.tsx
✅ user-avatar.tsx
✅ dashboard-header.tsx

❌ HomeNavbar.tsx
❌ UserAvatar.tsx
❌ DashboardHeader.tsx
```

The exported component name is still PascalCase — only the filename is kebab-case:
```tsx
// home-navbar.tsx
export function HomeNavbar() { ... }
```

---

**No hardcoded color values — ever:**
```tsx
// ❌ Hardcoded hex / RGB / Tailwind palette color
<div className="bg-[#1a1a1a] text-white border-gray-200">
<p className="text-red-500">Error</p>
<div className="bg-slate-900">

// ✅ Semantic CSS variable tokens from your globals.css
<div className="bg-background text-foreground border-border">
<p className="text-destructive">Error</p>
<div className="bg-card text-card-foreground">
```

The full token set from shadcn/tweakcn:

| Token | Use |
|---|---|
| `background` / `foreground` | Page background and primary text |
| `card` / `card-foreground` | Card surfaces |
| `primary` / `primary-foreground` | Brand colour buttons and highlights |
| `secondary` / `secondary-foreground` | Secondary actions |
| `muted` / `muted-foreground` | Subtle backgrounds and placeholder text |
| `accent` / `accent-foreground` | Hover states |
| `destructive` / `destructive-foreground` | Errors and danger actions |
| `border` | All borders |
| `input` | Input borders |
| `ring` | Focus rings |

If you catch yourself reaching for `text-gray-500` — use `text-muted-foreground`. For `text-red-500` — use `text-destructive`. For `bg-white` — use `bg-background`.

---

**No hardcoded `px` values — use the Tailwind scale and relative units:**
```tsx
// ❌ Arbitrary pixel values
<div className="p-[14px] mt-[32px] w-[320px] text-[13px]">
<div style={{ marginTop: "24px", fontSize: "14px" }}>

// ✅ Tailwind spacing scale or relative units
<div className="p-3.5 mt-8 w-80 text-sm">
```

**Width and height — prefer relative over fixed:**
```tsx
// ❌ Fixed pixel widths that break on different screens
<div className="w-[1200px]">
<div className="h-[600px]">

// ✅ Relative / responsive
<div className="w-full max-w-5xl">          // constrained but fluid
<div className="h-screen">                  // viewport-relative
<div className="min-h-[50vh]">              // viewport units OK for layout anchors
<div className="w-full md:w-1/2 lg:w-1/3">  // responsive columns
```

**Font sizes — use Tailwind text scale only:**
```tsx
// ❌
<p className="text-[13px]">
<h1 style={{ fontSize: "32px" }}>

// ✅
<p className="text-sm">    // 14px
<h1 className="text-3xl">  // 30px
```

**Spacing scale (multiples of 4px):**
```
1 = 4px   2 = 8px   3 = 12px   4 = 16px   5 = 20px   6 = 24px
8 = 32px  10 = 40px  12 = 48px  16 = 64px  20 = 80px  24 = 96px
```

Arbitrary values like `p-[14px]` are only acceptable when the design spec requires a value off the scale — and even then, document why.

**Responsive design — mobile-first:**
```tsx
// ✅ Start with mobile layout, expand for larger screens
<div className="flex flex-col gap-4 md:flex-row md:gap-6">
<div className="text-base md:text-lg lg:text-xl">
<div className="px-4 md:px-8 lg:px-16">
```

Never hard-code a layout for desktop only and add `hidden sm:block` as an afterthought.

---

**`page.tsx` files stay server components — never add `"use client"` to them:**

```tsx
// ❌ Slapping "use client" on the page kills SSR.
//    The crawler sees an empty shell + a hydration script. Bad for SEO,
//    bad for initial paint, bad for sharing previews (OG tags).
"use client";

export default function ProductPage() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

```tsx
// ✅ Page stays server-rendered. Interactivity lives in a child component.
//    The product title, description, price etc. all reach the crawler.

// src/app/products/[id]/page.tsx
import { getProduct } from "@/features/products/queries";
import { AddToCartButton } from "./_components/add-to-cart-button";

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const product = await getProduct(id);
  return (
    <>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      <AddToCartButton productId={product.id} />
    </>
  );
}

// src/app/products/[id]/_components/add-to-cart-button.tsx
"use client";
import { useState } from "react";
import { Button } from "@/components/ui/button";

export function AddToCartButton({ productId }: { productId: string }) {
  const [pending, setPending] = useState(false);
  // ...
  return <Button disabled={pending}>Add to cart</Button>;
}
```

**Rule of thumb:** if the file is `page.tsx`, `layout.tsx`, or anything Next.js conventionally treats as a route entry, it stays a server component. Push interactivity (event handlers, state, effects) into a child marked `"use client"` and import it in.

---

### 17. STANDARDS.md

Write `STANDARDS.md` in full. Do not stub.

```markdown
# STANDARDS.md

Engineering rules for this repo. Every code change should satisfy them.

## Architecture (keep it modular)

- Feature folders: `schema → queries → actions → index → components`
- Cross-feature imports go through `index.ts`. Never deep-import.
- Queries read. Actions write (auth → safeParse → `getDb()` write → log → audit → revalidate → `ok`).
- `"use server"` only at the **top** of `actions.ts` files, never inline in JSX.
- `page.tsx` / `layout.tsx` never use `"use client"`.
- Server components import `queries.ts` directly.
- Do not add `/api/v1` mutations. If you need a public HTTP API, use the REST scaffold.

## Errors, logging, observability (four layers — do not mix)

1. Expected failure = `fail("…")`. Crash = thrown `Error`. `withAction` is the only catch.
2. Client sees `ActionResult` only. Never `err.message` or a stack.
3. Pino: `log.<level>({ ...context }, "feature.verb")`. No `console.*`.
4. Writes also call `logAudit`. Reads do neither. IP on `audit_logs` + client-error / `request.error` only.

- Dual `level` + `severity`. `APP_ENV` drives source maps, not `NODE_ENV`.
- `/api/health` does `SELECT 1`. Redis is rate limit only — fail open. No Redis/CDN cache of authenticated HTML/JSON. No Sentry/PostHog without a DSN.

## Auth, secrets, webhooks, rate limit

- Auth is the first line of every action.
- Webhooks: HMAC on **raw body**, then parse. Idempotent. Not rate-limited.
- Rate limit unauthenticated public HTTP by IP. Fail open if Redis is missing.
- No top-level `throw` on runtime env — `getDb()`, `databaseUrl()`.
- No secret in code, committed env, or `NEXT_PUBLIC_*`.

## UI, tests, git

- kebab-case files. Tokens + Tailwind scale. shadcn only.
- Forms: `react-hook-form` + `zodResolver` + the same schema the action `safeParse`s.
- Test every action: happy path + auth-fail + validation-fail. Assert log event names on writes.
- `revalidatePath` after every mutation.
- Commit: `type: description`. Comments explain why, not ticket numbers.
```

---

### 18. CLAUDE.md

Write `CLAUDE.md` in full. Do not stub. Nested code fences are forbidden inside this template — use indented trees.

```markdown
# CLAUDE.md

Companion files:
- [`AGENTS.md`](./AGENTS.md) — router + non-negotiables
- [`STANDARDS.md`](./STANDARDS.md) — engineering rules
- [`REVIEW.md`](./REVIEW.md) — review rubric

Never commit, push, branch, or open a PR unless the user explicitly asks. Never `--no-verify`.

## Repo orientation

Next.js 16. Bun = package manager, Node.js = runtime. Mutations are **server actions**, not `/api/v1`.

| Surface | Where |
|---|---|
| Authenticated UI | `src/app/(platform)/` |
| Public pages | `src/app/(auth)/`, terms, privacy |
| Auth HTTP | `src/app/api/auth/[...nextauth]/` |
| Webhooks | `src/app/api/webhooks/<vendor>/` |
| Health | `src/app/api/health/` |
| Client error ingest | `src/app/api/log/client-error/` |
| Feature | `src/features/<name>/` — schema, queries, actions, components |
| Shared | `src/lib/` — `with-action`, `log`, `audit`, `getDb` via `@/db` |

## Canonical feature

        src/features/<name>/
          schema.ts     Zod
          queries.ts    reads
          actions.ts    "use server" + withAction
          index.ts      barrel
          components/

Action pipeline: withAction → auth → safeParse → getDb() write → log.info → logAudit → revalidatePath → ok().
Cross-feature via `index.ts`. Client/edge must sub-path import, not the barrel (pg in the barrel).

## Four layers

        fail() / throw Error → withAction → ActionResult (user) + log (operator)
        log.info / warn / error → stdout JSON
        logAudit → audit_logs

IP on audit + client-error / request.error only. Redis = rate limit, fail open, not a cache. No Sentry/PostHog without a DSN.

## Playbooks

### Add a feature
Run `bun run create-feature <kebab-name>`. Then DB table, AuditAction, uncomment page export, migrate.

### Add a server action
1. Schema in `schema.ts`
2. `export async function x(input: unknown) { return withAction("feature.verb", async () => { ... }); }`
3. Re-export from `index.ts`
4. Test happy + auth-fail + validation-fail

### Add a public REST API
Don't. Use the REST `/api/v1` scaffold. This app's HTTP surface is auth + webhooks + health + client-error.

### Add a webhook
`runtime = "nodejs"`. HMAC on raw body (`req.text()`), then parse, then Zod. Idempotent. Do not rate-limit.

### Add an env var
Lazy getter. `.env.example`. `instrumentation.ts` warn-list if required. Not `NEXT_PUBLIC_*` if secret. `APP_ENV` is a build ARG.

## Gotchas

- `page.tsx` is never `"use client"`.
- `"use server"` files cannot re-export types.
- `getDb()` is lazy. Top-level `new Pool(process.env.DATABASE_URL)` dies in `next build`.
- `audit_logs.user_id` is FK → users.id, nullable. System events pass `null`.
- Do not cache authenticated pages in Redis.

## Commands

| Command | What |
|---|---|
| `bun install` | Install deps |
| `bun run dev` | Dev :3000 |
| `bun run build` | Production build |
| `bun run create-feature <name>` | Scaffold a feature |
| `bunx vitest run` | Tests |
| `bunx tsc --noEmit` | Typecheck |
| `bunx next lint` | ESLint |
| `bunx drizzle-kit generate` | Generate migration |
| `bunx drizzle-kit migrate` | Apply migrations |
| `bun audit --audit-level=high` | Security audit |
```


---

### 19. AGENTS.md

Write `AGENTS.md` in full. It is the **router**.

```markdown
# AGENTS.md

## Coding principles

- YAGNI. Prefer `withAction`, `fail`/`ok`, `log`, `logAudit`, `getDb()`.
- Comments explain the code's why, not ticket numbers.

## Work management

- Never commit, push, branch, or run `gh pr create` unless the user explicitly asks. Never `--no-verify`.

**This file is a router.**

| Before you… | Read |
|---|---|
| Write an action, query, schema, or feature | [`STANDARDS.md`](./STANDARDS.md) |
| Review a diff | [`REVIEW.md`](./REVIEW.md) |
| Orient / add a feature | [`CLAUDE.md`](./CLAUDE.md) |

## Non-negotiables

1. Every mutation is a server action wrapped in `withAction`. That is the only catch. Return `ActionResult`. Never leak `err.message`.
2. Auth is the first line inside the action. Other users' ids → not-found, not forbidden.
3. `"use server"` only at the top of `actions.ts`. No `/api/v1` mutations.
4. Pino only. Writes call `log.info` + `logAudit`. GETs do neither.
5. Webhooks verify HMAC on the raw body before parse. Idempotent.
6. No secret in code / `NEXT_PUBLIC_*`. No module-top-level env throw — `getDb()`.
7. Rate limit public HTTP by IP; fail open if Redis is missing. Redis is not a response cache.
8. IP is personal data: audit + client-error / `onRequestError` only.
9. `APP_ENV` drives source maps. `NODE_ENV` does not.
10. Feature folders stay split. Cross-feature via `index.ts`.

## Review guidelines

### Block-merge

- `"use server"` inline in JSX
- Action not wrapped in `withAction` / not returning `ActionResult`
- Webhook `req.json()` before HMAC
- Module-load throws for env vars
- `NOT NULL` on populated table without default
- Hardcoded secrets
- `console.*` in app code
- Redis/CDN cache of authenticated pages
- Thrown `err.message` across the action boundary

### Discuss

- Missing `logAudit` or `revalidatePath` on a write
- New `AuditAction` not in the union
- Rate limit keyed by user id alone
- IP on every `log.info`
- `"use client"` on `page.tsx`

### Skip

- Prettier nits
- "Add Sentry / PostHog / Redis response cache" with no sink

## Confidence calibration

- Read the file, not just the diff
- Match existing patterns
- If unsure, don't flag unless block-merge or discuss
```

---

### 20. REVIEW.md

Write `REVIEW.md` in full. Keep in sync with `AGENTS.md` and `STANDARDS.md`.

```markdown
# REVIEW.md

Read [`CLAUDE.md`](./CLAUDE.md) and [`STANDARDS.md`](./STANDARDS.md) first.

## Block-merge

- `"use server"` outside an `actions.ts` file top
- Missing `withAction` / `ActionResult`
- Webhook signature missing or `req.json()` first
- Module-load env throws
- `NOT NULL` without default on populated table
- FK: non-user UUID in `audit_logs.user_id`
- `console.*`; stacks in user-facing errors
- Redis/CDN cache of authenticated HTML/JSON

## Discuss

- Missing `invalidate`/`revalidatePath` or `logAudit` on writes
- Wrong `fail()` copy; leaking internals
- IP on every log line
- Hardcoded color/px; PascalCase filenames
- Forms without `react-hook-form` + `zodResolver`

## Skip

- Style nits, theoretical races, "add Sentry/PostHog"

## Schema PRs

Block: schema/SQL drift, NOT NULL without default, dropping live columns.
Discuss: unused indexes, enum drift.

## Logging PRs

Block: `console.*`, missing `{ err }` on failures, logging tokens.
Discuss: event name not `feature.verb`; write without `logAudit`.
```

---

### 21. PR template

Create `.github/pull_request_template.md`:

```markdown
## Summary

<!-- 1-3 bullets: what changed and why -->

-

## Checklist

- [ ] Followed [STANDARDS.md](../STANDARDS.md)
- [ ] Tests pass locally (`bunx vitest run`)
- [ ] Typecheck + lint pass (`bunx tsc --noEmit`, `bunx next lint`)
- [ ] Writes use `withAction` + `log.info` + `logAudit` + `revalidatePath`
- [ ] If adding a webhook: HMAC on raw body, then parse; idempotent
- [ ] No secrets in code / `NEXT_PUBLIC_*`; no module-top-level env throws

## Test plan

<!-- How to verify this works -->

-
```

---

### 22. `next.config.ts`

`APP_ENV` is a **build ARG** (`local` | `dev` | `prod`). Unset/garbage → `local`. Prod keeps source maps **off**.
```typescript
import type { NextConfig } from "next";
import { ENVIRONMENT_SETTINGS, appEnvironment } from "./src/lib/app-environment";

const environment = ENVIRONMENT_SETTINGS[appEnvironment(process.env.APP_ENV)];

const nextConfig: NextConfig = {
  output: "standalone",
  productionBrowserSourceMaps: environment.enableBrowserSourceMaps,
  serverExternalPackages: ["pg", "pino"],
  transpilePackages: [],
  images: {
    remotePatterns: [{ protocol: "https", hostname: "lh3.googleusercontent.com" }],
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

### 23. `instrumentation.ts`

```typescript
export async function register(): Promise<void> {
  if (process.env.NEXT_RUNTIME !== "nodejs") return;

  const { attachProcessHandlers } = await import("./lib/process-handlers");
  attachProcessHandlers();

  const { log } = await import("./lib/log");
  const required = ["DATABASE_URL", "AUTH_SECRET", "NEXT_PUBLIC_APP_URL"] as const;
  for (const key of required) {
    if (!process.env[key]) log.warn({ key }, "instrumentation.missing_env");
  }
}

/**
 * Called by Next.js for every uncaught error during a request.
 * The `digest` matches what the user sees in error.tsx, so support
 * tickets can be cross-referenced with log lines.
 */
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

### 24. Error + loading pages

**`src/app/error.tsx`** — catches render errors inside any route:
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
    reportClientError(error, "error.tsx");
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

**`src/app/global-error.tsx`** — catches errors in the root layout itself:
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

**`src/app/(platform)/loading.tsx`** — shown during server component suspense:
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

### 25. Rate limiting

IP-based sliding window using Upstash Redis. Works across multiple instances, runs on both edge (middleware) and Node.js (route handlers).

**Upstash setup (2 minutes):**
1. Go to [console.upstash.com](https://console.upstash.com) → Create Database
2. Name it, pick the region closest to your app
3. Plan: **Free** tier (10,000 req/day) is enough for dev and low-traffic prod. For production use **Pay as you go** — rate limit calls are tiny (~1 req per inbound HTTP request), costs stay near zero unless you're handling millions of requests/day
4. Go to **REST API** tab → copy both values into `.env.local`

**`src/lib/rate-limit.ts`** — lazy Redis. Missing Upstash → fail **open** + warn once. Never `Redis.fromEnv()` at import.
```typescript
import { Ratelimit } from "@upstash/ratelimit";
import { Redis } from "@upstash/redis";
import { log } from "@/lib/log";

export type RateLimitResult = { allowed: true } | { allowed: false; retryAfter: number };

function getRedis(): Redis | null {
  const url = process.env.UPSTASH_REDIS_REST_URL;
  const token = process.env.UPSTASH_REDIS_REST_TOKEN;
  if (!url || !token) return null;
  return new Redis({ url, token });
}

export function createRateLimiter(config: { windowMs: number; max: number }) {
  let warned = false;
  return async function check(ip: string): Promise<RateLimitResult> {
    const redis = getRedis();
    if (!redis) {
      if (!warned) {
        warned = true;
        log.warn({}, "ratelimit.disabled_no_redis");
      }
      return { allowed: true };
    }
    try {
      const limiter = new Ratelimit({
        redis,
        limiter: Ratelimit.slidingWindow(config.max, `${config.windowMs}ms`),
        prefix: `rl:${config.windowMs}:${config.max}`,
      });
      const { success, reset } = await limiter.limit(ip);
      if (!success) return { allowed: false, retryAfter: Math.ceil((reset - Date.now()) / 1000) };
      return { allowed: true };
    } catch (err) {
      log.warn({ err }, "ratelimit.redis_error_fail_open");
      return { allowed: true };
    }
  };
}
```

**Usage — route handler:**
```typescript
import { NextRequest, NextResponse } from "next/server";
import { createRateLimiter } from "@/lib/rate-limit";
import { getIp } from "@/lib/get-ip";

const limiter = createRateLimiter({ windowMs: 15 * 60 * 1000, max: 5 });

export async function POST(req: NextRequest) {
  const result = await limiter(getIp(req));
  if (!result.allowed) {
    return NextResponse.json(
      { error: "Too many requests. Please try again later." },
      { status: 429, headers: { "Retry-After": String(result.retryAfter) } }
    );
  }
  // ... handler logic
}
```

**Sensible defaults per route type:**

| Route | Window | Max | Why |
|---|---|---|---|
| Auth (login, signup) | 15 min | 10 | Brute-force protection |
| Contact / waitlist form | 15 min | 5 | Spam prevention |
| Public API (unauthenticated) | 1 min | 60 | General abuse prevention |
| Password reset | 1 hour | 3 | Account enumeration protection |
| Client error reporting | 1 min | 10 | Log-bill flood prevention |

**Rules:**
- Always use IP as the key — never user ID alone
- Apply to unauthenticated public HTTP routes — webhooks and `/api/health` are exempt
- Fail **open** if Redis is missing or throws
- Redis is **not** a page/JSON cache. After writes, `revalidatePath`.
- Always return `Retry-After`
- `"unknown"` IPs share one bucket

---

### 25b. Webhooks — raw body, then parse

Canonical handler. **Never** `req.json()` before verify.

**`src/lib/webhook.ts`:**
```typescript
import { createHmac, timingSafeEqual } from "node:crypto";
import { webhookSecret } from "@/lib/env";

export function verifyHmacSha256(rawBody: string, header: string | null): boolean {
  const secret = webhookSecret();
  if (!secret || !header) return false;
  const digest = createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(digest);
  const b = Buffer.from(header);
  return a.length === b.length && timingSafeEqual(a, b);
}
```

**`src/app/api/webhooks/stripe/route.ts`:**
```typescript
import { NextRequest, NextResponse } from "next/server";
import { z } from "zod";
import { log } from "@/lib/log";
import { verifyHmacSha256 } from "@/lib/webhook";

export const runtime = "nodejs";

const EventSchema = z.object({
  id: z.string().max(200),
  type: z.string().max(100),
});

export async function POST(req: NextRequest) {
  const raw = await req.text();
  if (!verifyHmacSha256(raw, req.headers.get("x-webhook-signature"))) {
    log.warn({}, "webhooks.signature_rejected");
    return NextResponse.json({ error: "Invalid signature." }, { status: 401 });
  }
  let json: unknown;
  try {
    json = JSON.parse(raw);
  } catch {
    return NextResponse.json({ error: "Invalid payload." }, { status: 400 });
  }
  const parsed = EventSchema.safeParse(json);
  if (!parsed.success) {
    return NextResponse.json({ error: "Invalid payload." }, { status: 400 });
  }
  log.info({ eventId: parsed.data.id, type: parsed.data.type }, "webhooks.received");
  return NextResponse.json({ received: true });
}
```

---

### 26. CI workflow

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
    timeout-minutes: 20
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
    timeout-minutes: 20
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
      - name: Cache deps
        uses: actions/cache@v4
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

**Key design decisions:**
- `lint`, `format`, `typecheck`, `audit`, `test` run **in parallel** — build only starts after all pass
- `cancel-in-progress` kills stale runs on rapid pushes but never on `main`
- `--frozen-lockfile` — CI fails if `bun.lock` is out of sync with `package.json`
- Build job sets dummy env vars so `next build`'s page-data collection doesn't throw on missing vars

---

### 27. Email — Nodemailer + Gmail

Transactional email via Gmail SMTP. Free, no third-party signup, works for low-volume apps (password resets, verification emails, notifications). For high-volume production, swap the transport for Resend / Postmark / SES — the `sendEmail` API stays identical.

**Gmail App Password setup (one-time):**
1. The Gmail account must have **2-Step Verification** enabled — [myaccount.google.com/security](https://myaccount.google.com/security)
2. Go to [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords)
3. Generate a new App Password (regular Gmail passwords don't work for SMTP)
4. Copy the 16-char password and put it in `.env.local` as `GMAIL_APP_PASSWORD`

**Install:**
```bash
bun add nodemailer
bun add -d @types/nodemailer
```

Add to `.env.example`:
```bash
# Email (Gmail SMTP via App Password)
GMAIL_USER=your-app@gmail.com
GMAIL_APP_PASSWORD=xxxxxxxxxxxxxxxx
EMAIL_FROM_NAME={{APP_NAME}}
```

**`src/lib/email.ts`:**
```typescript
import nodemailer, { type Transporter } from "nodemailer";
import { log } from "@/lib/log";

let cachedTransporter: Transporter | null = null;

function getTransporter(): Transporter {
  if (cachedTransporter) return cachedTransporter;

  const user = process.env.GMAIL_USER;
  const pass = process.env.GMAIL_APP_PASSWORD;
  if (!user || !pass) {
    throw new Error("GMAIL_USER and GMAIL_APP_PASSWORD are required for email");
  }

  cachedTransporter = nodemailer.createTransport({
    service: "gmail",
    auth: { user, pass },
  });
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
    log.info(
      { to: params.to, subject: params.subject, messageId: info.messageId },
      "email.sent"
    );
    return { ok: true, messageId: info.messageId };
  } catch (err) {
    log.error({ to: params.to, subject: params.subject, err }, "email.send_failed");
    return { ok: false, error: err instanceof Error ? err.message : "Unknown error" };
  }
}
```

**Usage from a server action:**
```typescript
"use server";

import { sendEmail } from "@/lib/email";

export async function sendWelcomeEmail(userId: string, email: string) {
  const result = await sendEmail({
    to: email,
    subject: "Welcome to {{APP_NAME}}",
    html: `<h1>Hi there</h1><p>Thanks for signing up.</p>`,
    text: "Hi there. Thanks for signing up.",
  });

  if (!result.ok) {
    // Don't fail the user-facing flow on email errors — log and continue
    return;
  }
}
```

**Rules:**
- **Always send `text` alongside `html`** — some mail clients still prefer plain text, and missing it hurts deliverability scores
- **Never `await sendEmail` in the critical path** — if Gmail's API is slow, the user waits. Fire-and-forget for non-blocking emails (welcome, notifications); reserve `await` only when the user explicitly triggered "send me the email" and waits for confirmation
- **Use `replyTo` for support emails** — set `replyTo: "support@yourdomain.com"` so replies don't go to the Gmail inbox
- **Gmail SMTP limit: 500 messages/day for free accounts, 2000/day for Workspace** — past that, switch to Resend / Postmark
- **Never put secrets in email body** — assume mail can be forwarded. Send a link to a server-rendered page instead

---

### 28. SEO + assets

#### Root-layout metadata

**`src/app/layout.tsx`** — sets the global SEO defaults that every page inherits:
```tsx
import type { Metadata } from "next";
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
      <body>{children}</body>
    </html>
  );
}
```

The `template: "%s | {{APP_NAME}}"` means any page that sets `title: "Pricing"` renders as `"Pricing | {{APP_NAME}}"` — consistent suffix everywhere.

#### Per-page metadata

Pages override the defaults via `generateMetadata` for dynamic content:
```tsx
// src/app/products/[id]/page.tsx
import type { Metadata } from "next";
import { getProduct } from "@/features/products/queries";

export async function generateMetadata({
  params,
}: {
  params: Promise<{ id: string }>;
}): Promise<Metadata> {
  const { id } = await params;
  const product = await getProduct(id);
  return {
    title: product.name,
    description: product.shortDescription,
    openGraph: {
      title: product.name,
      description: product.shortDescription,
      images: [product.imageUrl ?? "/opengraph-image"],
    },
  };
}
```

#### `sitemap.ts`

**`src/app/sitemap.ts`** — Next.js auto-serves this at `/sitemap.xml`:
```typescript
import type { MetadataRoute } from "next";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const staticRoutes: MetadataRoute.Sitemap = [
    { url: APP_URL, lastModified: new Date(), changeFrequency: "daily", priority: 1.0 },
    { url: `${APP_URL}/login`, lastModified: new Date(), changeFrequency: "monthly", priority: 0.3 },
    { url: `${APP_URL}/terms`, lastModified: new Date(), changeFrequency: "yearly", priority: 0.2 },
    { url: `${APP_URL}/privacy`, lastModified: new Date(), changeFrequency: "yearly", priority: 0.2 },
  ];

  // For dynamic content (blog posts, products, etc.), fetch from DB and map:
  // const products = await db.query.products.findMany({ columns: { slug: true, updatedAt: true } });
  // const dynamicRoutes = products.map((p) => ({
  //   url: `${APP_URL}/products/${p.slug}`,
  //   lastModified: p.updatedAt,
  //   changeFrequency: "weekly" as const,
  //   priority: 0.7,
  // }));

  return staticRoutes;
}
```

#### `robots.ts`

**`src/app/robots.ts`** — Next.js auto-serves this at `/robots.txt`:
```typescript
import type { MetadataRoute } from "next";

const APP_URL = process.env.NEXT_PUBLIC_APP_URL ?? "http://localhost:3000";

export default function robots(): MetadataRoute.Robots {
  return {
    rules: [
      {
        userAgent: "*",
        allow: "/",
        disallow: ["/api/", "/dashboard", "/settings"],
      },
    ],
    sitemap: `${APP_URL}/sitemap.xml`,
    host: APP_URL,
  };
}
```

#### Favicon + OG image conventions

Next.js looks for these files in `src/app/` by filename (no manual `<link>` tags needed):

| Filename | Purpose | Recommended size |
|---|---|---|
| `icon.png` | Browser tab favicon | 32×32 or 512×512 |
| `apple-icon.png` | iOS home-screen icon | 180×180 |
| `opengraph-image.png` | Default OG image (Facebook, LinkedIn, Slack previews) | 1200×630 |
| `twitter-image.png` | Twitter card image | 1200×600 |

**Default = same as favicon:** generate `opengraph-image.png` from your favicon — upscale the favicon to fit within 1200×630, centered on a background that matches your brand color from tweakcn.

**Tools:**
- [realfavicongenerator.net](https://realfavicongenerator.net) — drop one source image, get all sizes
- Figma or any image editor — 1200×630 canvas, drop logo in center, export PNG

#### Public folder — asset management rules

**All static assets live in `public/`.** Reference them with absolute paths from the URL root:
```tsx
// ✅ Absolute path — resolves to /public/logo.svg
<img src="/logo.svg" alt="Logo" />
<Image src="/images/hero.jpg" alt="Hero" width={1200} height={600} />

// ❌ Relative path — breaks under different route depths
<img src="./logo.svg" />
<img src="../public/logo.svg" />
```

**Recommended subfolder structure:**
```
public/
  images/          ← photos, illustrations
  icons/           ← custom SVG icons not in your icon package
  fonts/           ← only if NOT using next/font (prefer next/font)
  videos/          ← MP4s, WebM
  documents/       ← PDFs the user can download
```

**Rules:**
- **Never put secrets, API keys, or private docs in `public/`** — every file is served unconditionally at its path, no auth check
- **Never put large files (>1MB) in `public/`** for marketing apps — they bloat the deploy and slow first paint. Use a CDN or object storage (Cloudflare R2, S3) for big assets
- **Prefer `next/font` over self-hosted font files** — automatic preloading, no FOUT, zero CLS. Only put fonts in `public/fonts/` if a vendor license forbids next/font's processing
- **Use `next/image` for raster images** — automatic responsive sizing, lazy loading, WebP/AVIF conversion. Plain `<img>` skips all of that
- **SVGs are fine as inline React components or in `public/icons/`** — for icons you reuse in code, inline components win (tree-shakable, dynamic styling); for one-off illustrations, `public/` is fine

---

### 29. Vercel deployment

The app deploys to Vercel via a GitHub Actions workflow on push to `main`. Vercel's automatic git integration is **disabled** so deployments are explicit and controlled by CI — same gating as everywhere else (lint + typecheck + test must pass first, conceptually, though this scaffold runs CI in parallel with deploy).

**One-time setup:**
1. Create the project on Vercel: `vercel link` from the repo root (or import via Vercel dashboard)
2. Get the project ID and org ID from `.vercel/project.json` (created by `vercel link`) or from Vercel dashboard → Project Settings → General
3. Create a Vercel API token at [vercel.com/account/tokens](https://vercel.com/account/tokens) — scope it to the project
4. Add three repo secrets in GitHub → Settings → Secrets and variables → Actions:
   - `VERCEL_TOKEN` — the API token
   - `VERCEL_ORG_ID` — starts with `team_` or `user_`
   - `VERCEL_PROJECT_ID` — starts with `prj_`

**`vercel.json`** — disables Vercel's automatic git deployments so only the workflow below triggers a deploy:
```json
{
  "git": {
    "deploymentEnabled": {
      "main": false
    }
  }
}
```

> Without this, every push to `main` triggers TWO deploys — one from Vercel's git integration, one from the workflow. Disabling the git side keeps the deploy source-of-truth in CI.

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
          test -n "${{ env.VERCEL_ORG_ID }}" || (echo "Missing VERCEL_ORG_ID (user_...)" && exit 1)
          test -n "${{ env.VERCEL_PROJECT_ID }}" || (echo "Missing VERCEL_PROJECT_ID (prj_...)" && exit 1)

      - name: Install Vercel CLI
        run: npm i -g vercel@latest

      # Pull env/linked settings for Production from Vercel project
      - name: Vercel pull (production)
        run: vercel pull --yes --environment=production --token ${{ secrets.VERCEL_TOKEN }}

      # Build locally using Vercel
      - name: Vercel build (production)
        run: vercel build --prod --token ${{ secrets.VERCEL_TOKEN }}

      # IMPORTANT: Remove git metadata to bypass "Git author must have access" checks
      - name: Remove Git metadata
        run: |
          rm -rf .git
          unset GITHUB_ACTOR GITHUB_SHA GITHUB_REF GITHUB_HEAD_REF GITHUB_REPOSITORY

      # Deploy the prebuilt artifacts to Production
      - name: Vercel deploy (production)
        run: vercel deploy --prebuilt --prod --yes --token ${{ secrets.VERCEL_TOKEN }}
```

**Environment variables on Vercel:**
Set every env var from `.env.example` in Vercel → Project Settings → Environment Variables → Production scope. The workflow's `vercel pull` step downloads them into the build so `next build` sees them. **Don't put secrets in the workflow file** — they live in Vercel's env-var store + GitHub Actions secrets.

---

### 30. Dockerfile + .dockerignore

For deploying outside Vercel (Cloud Run, Fly, Railway, self-hosted). Uses the `output: "standalone"` config from section 22 to produce a minimal runtime image.

**`Dockerfile`** — multi-stage build:
```dockerfile
# syntax=docker/dockerfile:1.6

# ── Stage 1: Install deps with Bun ─────────────────────────────────────────
FROM oven/bun:1 AS deps
WORKDIR /app
COPY package.json bun.lock ./
RUN bun install --frozen-lockfile

# ── Stage 2: Build the app ─────────────────────────────────────────────────
FROM oven/bun:1 AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

ARG APP_ENV=dev
ARG NEXT_PUBLIC_APP_URL
ENV APP_ENV=$APP_ENV
ENV NEXT_PUBLIC_APP_URL=$NEXT_PUBLIC_APP_URL
ENV NEXT_TELEMETRY_DISABLED=1

RUN bun run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ARG APP_ENV=dev
ENV APP_ENV=$APP_ENV
ARG NEXT_PUBLIC_APP_URL
ENV NEXT_PUBLIC_APP_URL=$NEXT_PUBLIC_APP_URL

# Non-root user for runtime
RUN addgroup -S nodejs && adduser -S nextjs -G nodejs

# Copy the standalone output produced by output: "standalone"
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["node", "server.js"]
```

**`.dockerignore`** — keep the build context small:
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

# Env files — pass secrets via build args, not COPY
.env
.env.local
.env.production
.env*.local

# Test + build artifacts
coverage
**/dist
**/build
**/.turbo

# Docs and metadata
README.md
CHANGELOG.md
LICENSE
*.md

# Drizzle meta (not needed at runtime — migrations run separately)
drizzle/meta

# IDE / OS junk
Thumbs.db
```

**Build + run locally:**
```bash
docker build \
  --build-arg DATABASE_URL=postgresql://... \
  --build-arg AUTH_SECRET=... \
  --build-arg AUTH_GOOGLE_ID=... \
  --build-arg AUTH_GOOGLE_SECRET=... \
  --build-arg NEXT_PUBLIC_APP_URL=https://your-app.com \
  --build-arg UPSTASH_REDIS_REST_URL=... \
  --build-arg UPSTASH_REDIS_REST_TOKEN=... \
  -t {{APP_NAME}} .

docker run -p 3000:3000 \
  -e DATABASE_URL=postgresql://... \
  -e AUTH_SECRET=... \
  -e AUTH_GOOGLE_ID=... \
  -e AUTH_GOOGLE_SECRET=... \
  -e NEXT_PUBLIC_APP_URL=https://your-app.com \
  -e UPSTASH_REDIS_REST_URL=... \
  -e UPSTASH_REDIS_REST_TOKEN=... \
  {{APP_NAME}}
```

**Why the runner stage uses Node and not Bun:** Next.js's standalone output bundles its own `server.js` which expects the Node.js runtime. Using Bun here works for many apps but hits edge cases with some Node-specific packages (the same `pg`/`@google-cloud/*`-style native deps that need `serverExternalPackages` in `next.config.ts`). Stick with Node for the runtime; use Bun for builds.

---

### 31. .gitignore additions

Ensure these are in `.gitignore`:
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

### 32. Final checks

Run in order:
```bash
bun install
bunx drizzle-kit generate
bunx tsc --noEmit
bunx next lint
bunx vitest run
bunx next build
```

All should pass before the first commit. `next build` must succeed with dummy `DATABASE_URL` (lazy `getDb()`). Confirm `productionBrowserSourceMaps` is false when `APP_ENV=prod`.

**Post-scaffold (human, not the model):** Neon, Google OAuth, Upstash Redis, Vercel tokens, optional tweakcn, `WEBHOOK_SECRET` when you add a vendor.

First commit:
```bash
git add .
git commit -m "chore: initial project setup"
```

---

## What this scaffold gives you

| Piece | What it does |
|---|---|
| Bun package manager | Fast installs. Next.js runtime stays Node.js |
| **Server actions + `withAction`** | One catch. `ActionResult` envelope. Auth first. No `/api/v1` mutations |
| Drizzle + Postgres | Lazy `getDb()`, TLS verify on |
| NextAuth v5 + Google | JWT sessions; adapter via `getDb()` proxy |
| Pino logger | JSON + GCP severity, `APP_LOG_FILE`, redaction, `no-console` |
| Audit trail | `log.info` + `logAudit` on writes. IP on audit, not every log line |
| Rate limit | IP middleware; Redis for this only; fail open. No Redis response cache |
| Webhooks | HMAC on raw body, then parse; idempotent |
| Client errors | Same-origin, Zod, rate-limited `/api/log/client-error` |
| Source maps | `APP_ENV` build ARG — on local/dev, **off prod** |
| Governance docs | `CLAUDE.md` · `AGENTS.md` · `STANDARDS.md` · `REVIEW.md` — written in full |
| Dockerfile | Bun→Node 22, `APP_ENV` + `NEXT_PUBLIC_*` only at build — no secrets in the image |
| shadcn/ui | CSS variables (tweakcn optional later) |
| Forms | `react-hook-form` + `zodResolver` + same schema as the action |
| Feature template | `schema → queries → actions → index → components` |

## PROMPT END
