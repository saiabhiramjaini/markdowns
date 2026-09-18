# Next.js REST API Scaffold Prompt

Copy-paste this entire prompt to scaffold a production-ready Next.js app with **strict REST APIs, zero server actions**.
Replace `{{APP_NAME}}` with your app name (kebab-case, e.g. `my-app`).

**Use this when:** the app has a frontend AND a backend, but you want every mutation to go through versioned HTTP endpoints (so a mobile client, third-party integrator, or non-TS client can consume the same API the web frontend uses). All form submits go to `/api/v1/*`; server components fetch data via direct query imports (no HTTP roundtrip for SSR); client components fetch via React Query hooks wrapping typed fetch wrappers.

**Use the full-stack server-actions prompt instead when:** the app is Next.js-only with no plan for external API consumers — server actions give you better DX with less ceremony.

**Use the frontend-only prompt instead when:** you have no first-party backend at all.

---

## PROMPT START

Scaffold a production-ready REST API + Next.js frontend app called `{{APP_NAME}}`. Follow every instruction exactly — don't add extras, don't skip steps.

**Write `CLAUDE.md`, `AGENTS.md`, `STANDARDS.md`, and `REVIEW.md` in full from the templates in this prompt.** They are the contract the generated repo lives by. Do not stub, summarize, or omit rules. They must match §0 (errors / logging / observability), the feature-folder split, rate limiting, and the no-Redis-response-cache rule.

Non-interactive CLIs only: `bunx shadcn@latest init -d`. Do not open tweakcn / Neon / Google / Upstash / Vercel in a browser — stub `.env.example` and leave a post-scaffold checklist. Ship a default `globals.css` token set (shadcn Slate + CSS variables).

---

### 0. Exception handling, errors, logging, observability (read first)

Four layers. Do not mix them.

```
throw ApiError / Error     →   withApi catch
        │                         │
        │                         ├─ known ApiError  → HTTP envelope (user-safe)
        │                         └─ unknown Error   → log.error + HTTP 500 generic
        │
log.info / warn / error    →   stdout JSON (+ APP_LOG_FILE). Operators only.
logAudit(...)              →   audit_logs table. Durable "who did what".
```

**Exceptions (control flow + crashes)**

- Handlers throw `ApiError(status, code, message)` for expected failures (401, 404, 409, 422, 429). That is not a crash; `withApi` turns it into the error envelope.
- Handlers throw (or let Drizzle throw) a real `Error` only when something is actually wrong. `withApi` logs it and returns `500 { error: { code: "internal_error", message: "Something went wrong." } }`. Never put `err.message` or a stack in the HTTP body.
- `try/catch` lives in **one place**: `withApi`. Route files do not add a second catch. Queries/mutations do not catch-and-swallow.
- Unique-key collisions are mapped to `ApiError(409)` in mutations (`pg` code `23505`). Empty PATCH bodies are `ApiError(400)`.
- Process crashes: `uncaughtException` → `log.fatal` in a Node-only `process-handlers.ts` (not inside `instrumentation.ts` — Next's Edge scanner will warn). `unhandledRejection` → `log.error`, do not `process.exit`.

**Error handling (what the client sees)**

Envelope is the entire public contract:

```json
{ "data": { } }                                  // 2xx with a body
{ "error": { "code": "not_found", "message": "Thing not found." } }   // 4xx/5xx
```

- 204 DELETE has no body.
- List endpoints: `{ "data": { "items": [], "next_cursor": null } }`.
- Codes: `unauthorized` `forbidden` `validation_error` `bad_request` `unsupported_media_type` `payload_too_large` `not_found` `conflict` `rate_limited` `internal_error`.
- Other users' ids → **404**, not 403 (do not leak existence).
- User-facing `message` is canned English. Internals go in the log.

**Logging (operators, not users)**

Pino only. `console.*` is an ESLint error. Signature: `log.<level>({ ...context }, "feature.verb")`.

| Level | When |
|---|---|
| `fatal` | Process is going down |
| `error` | Request/job actually failed (5xx, thrown Error, persistence failed) |
| `warn` | Recoverable (expected 4xx already returned, rate limiter down, retryable) |
| `info` | State change (`things.created`) |
| `debug` | Off in prod unless `LOG_LEVEL=debug` |

Every API line includes `requestId` + `userId` (and `tokenId` when a Bearer token was used). Failures include `{ err }` so the serializer keeps stack + cause. Never log secrets; the redactor is backup, not permission to log `authorization`.

`log.info` is **not** the audit trail. Writes also call `logAudit`.

**Observability (how you find a request later)**

- `x-request-id`: honor inbound or mint a UUID; echo on every `/api/v1` response; bind `log.child({ requestId })`. Authenticated responses also set `Cache-Control: private, no-store`.
- Dual fields on every line: `level` (Datadog) + `severity` (GCP Cloud Logging).
- `service` from `DD_SERVICE` (default `{{APP_NAME}}`). Sidecar tails `APP_LOG_FILE` if set; stdout always; never throw at import if the file path is dead.
- `APP_ENV=local|dev|prod` is a **build ARG**. `NODE_ENV` is `production` on every image and must not drive source maps. Browser source maps on for local/dev, **off for prod**.
- `/api/health` → `SELECT 1` → 503 if the DB is down (Cloud Run probe this, not `/`).
- Client errors POST to `/api/log/client-error` (same-origin, Zod, size cap, IP rate limit) and become `log.error(..., "client.error_reported")`.
- Next `onRequestError` logs render/route explosions as `request.error`.

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

**Step 1 — Init shadcn (non-interactive):**
```bash
bunx shadcn@latest init -d
```

Defaults (Default style, Neutral/Slate, CSS variables) are fine. Do not wait for a TTY.

**Step 2 — Add the base component set:**
```bash
bunx shadcn@latest add button input label textarea dialog dropdown-menu badge toast card separator skeleton tabs select checkbox radio-group switch tooltip popover sheet command avatar form
```

**Step 3 — Theme:** keep the shadcn `globals.css` token set (`background`, `foreground`, `destructive`, `muted-foreground`, …). Optionally replace later from tweakcn.com — not part of codegen.

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

/** Call from request handlers / queries — never at module import. */
export function getDb(): Db {
  if (cached) return cached;
  const url = databaseUrl();
  pool = new Pool({ connectionString: url, ssl: sslFor(url) });
  cached = drizzle(pool, { schema });
  return cached;
}

export * from "./schema";
```

Use `getDb().query` / `getDb().insert` everywhere the old code said `db.` (queries, mutations, audit, health).

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
    ssl: isLocal ? false : { rejectUnauthorized: true },
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
  session: { strategy: "jwt" }, // JWT only. Do not add a sessions table. Adapter below persists users/accounts.
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

Create `src/middleware.ts` — UI auth redirect, `/api/v1` CORS + OPTIONS + IP rate limit. **API auth is not here** — `withApi` → `requireApiAuth` (session cookie **or** Bearer token). Webhooks skip rate limit (signature is the gate).
```typescript
import NextAuth from "next-auth";
import { NextResponse } from "next/server";
import { authConfig } from "@/features/auth/lib/config";
import { corsHeaders, isAllowedOrigin } from "@/lib/api/cors";
import { createRateLimiter } from "@/lib/rate-limit";
import { getIp } from "@/lib/get-ip";

const { auth } = NextAuth(authConfig);
const apiLimiter = createRateLimiter({ windowMs: 60_000, max: 60 });

export default auth(async (req) => {
  const { pathname } = req.nextUrl;
  const isApiV1 = pathname.startsWith("/api/v1/");

  if (isApiV1 && req.method === "OPTIONS") {
    if (!isAllowedOrigin(req)) return new NextResponse(null, { status: 403 });
    return new NextResponse(null, { status: 204, headers: corsHeaders(req) });
  }

  if (isApiV1) {
    const result = await apiLimiter(getIp(req));
    if (!result.allowed) {
      return NextResponse.json(
        { error: { code: "rate_limited", message: "Too many requests." } },
        {
          status: 429,
          headers: { "Retry-After": String(result.retryAfter), ...corsHeaders(req) },
        }
      );
    }
  }

  const isLoggedIn = !!req.auth;
  const isAuthRoute = pathname.startsWith("/api/auth");
  const isWebhook = pathname.startsWith("/api/webhooks");
  const isPublicPage = ["/login", "/terms", "/privacy"].includes(pathname);

  if (isAuthRoute || isWebhook || isPublicPage) return;
  // Browser cookie or Bearer — decided in the route. Do not redirect API clients to /login.
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

**`src/lib/env.ts`** — lazy getters. Call from a request, never at module import.
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

export function corsOrigins(): string[] {
  return (process.env.CORS_ORIGINS ?? process.env.NEXT_PUBLIC_APP_URL ?? "")
    .split(",")
    .map((s) => s.trim())
    .filter(Boolean);
}

export function webhookSecret(): string | undefined {
  return process.env.WEBHOOK_SECRET;
}
```

**`src/lib/app-environment.ts`** — build-time + runtime marker. `NODE_ENV` cannot tell Cloud Run `dev` from `prod`.
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

export function getActiveEnvironmentSettings(): EnvironmentSettings {
  return ENVIRONMENT_SETTINGS[appEnvironment(process.env.APP_ENV)];
}
```

**`src/lib/api/errors.ts`** — thrown by handlers; `withApi` is the only catcher. `retryAfter` is for 429.
```typescript
export class ApiError extends Error {
  constructor(
    public status: number,
    public code: string,
    message: string,
    public retryAfter?: number
  ) {
    super(message);
    this.name = "ApiError";
  }
}

export function isPgUniqueViolation(err: unknown): boolean {
  return typeof err === "object" && err !== null && "code" in err && err.code === "23505";
}
```

**`src/lib/api/response.ts`:**
```typescript
import { NextResponse } from "next/server";
import { log } from "@/lib/log";
import { ApiError } from "./errors";

export function apiOk<T>(data: T, status = 200): NextResponse {
  return NextResponse.json({ data }, { status });
}

export function apiNoContent(): NextResponse {
  return new NextResponse(null, { status: 204 });
}

export function apiError(err: unknown, extra?: { requestId?: string }): NextResponse {
  if (err instanceof ApiError) {
    const headers: Record<string, string> = {};
    if (err.retryAfter !== undefined) headers["Retry-After"] = String(err.retryAfter);
    if (extra?.requestId) headers["x-request-id"] = extra.requestId;
    // 4xx is expected; warn so it does not page like a 500.
    if (err.status >= 500) log.error({ err, requestId: extra?.requestId }, "api.unhandled_error");
    else log.warn({ status: err.status, code: err.code, requestId: extra?.requestId }, "api.rejected");
    return NextResponse.json({ error: { code: err.code, message: err.message } }, { status: err.status, headers });
  }
  log.error({ err, requestId: extra?.requestId }, "api.unhandled_error");
  return NextResponse.json(
    { error: { code: "internal_error", message: "Something went wrong." } },
    { status: 500, headers: extra?.requestId ? { "x-request-id": extra.requestId } : undefined }
  );
}

export function firstError(issues: { message: string }[], fallback: string): string {
  return issues[0]?.message ?? fallback;
}
```

**`src/lib/api/parse.ts`** — JSON body + query. Throws `ApiError`; do not catch in the route.
```typescript
import { z } from "zod";
import { ApiError } from "./errors";
import { firstError } from "./response";

const MAX_JSON_BYTES = 1_000_000;

export async function parseBody<T>(req: Request, schema: z.ZodType<T>): Promise<T> {
  const contentType = req.headers.get("content-type") ?? "";
  if (!contentType.toLowerCase().startsWith("application/json")) {
    throw new ApiError(415, "unsupported_media_type", "Content-Type must be application/json.");
  }
  const declared = Number(req.headers.get("content-length") ?? 0);
  if (declared > MAX_JSON_BYTES) {
    throw new ApiError(413, "payload_too_large", "Request body is too large.");
  }
  const text = await req.text();
  if (text.length > MAX_JSON_BYTES) {
    throw new ApiError(413, "payload_too_large", "Request body is too large.");
  }
  let json: unknown;
  try {
    json = JSON.parse(text);
  } catch {
    throw new ApiError(400, "bad_request", "Invalid JSON.");
  }
  const parsed = schema.safeParse(json);
  if (!parsed.success) {
    throw new ApiError(400, "validation_error", firstError(parsed.error.issues, "Invalid input"));
  }
  return parsed.data;
}

export function parseQuery<T>(url: string, schema: z.ZodType<T>): T {
  const parsed = schema.safeParse(Object.fromEntries(new URL(url).searchParams));
  if (!parsed.success) {
    throw new ApiError(400, "validation_error", firstError(parsed.error.issues, "Invalid query"));
  }
  return parsed.data;
}

export const UuidParamSchema = z.string().uuid({ message: "Invalid id" });
```

**`src/lib/api/cors.ts`** — allowlist only. Never `*`. Same-origin Next UI does not need CORS; mobile / extra origins do.
```typescript
import { corsOrigins } from "@/lib/env";

export function isAllowedOrigin(req: Request): boolean {
  const origin = req.headers.get("origin");
  if (!origin) return true; // same-origin or non-browser
  return corsOrigins().includes(origin);
}

export function corsHeaders(req: Request): Record<string, string> {
  const origin = req.headers.get("origin");
  if (!origin || !corsOrigins().includes(origin)) return {};
  return {
    "Access-Control-Allow-Origin": origin,
    "Access-Control-Allow-Methods": "GET,POST,PATCH,PUT,DELETE,OPTIONS",
    "Access-Control-Allow-Headers": "content-type,authorization,x-request-id",
    "Access-Control-Max-Age": "86400",
    Vary: "Origin",
  };
}

export function applyCors(req: Request, res: Response): Response {
  const headers = corsHeaders(req);
  for (const [key, value] of Object.entries(headers)) res.headers.set(key, value);
  return res;
}
```

**`src/lib/api/auth.ts`** — session cookie **or** `Authorization: Bearer`. Same handlers for web, mobile, curl.
```typescript
import { createHash, timingSafeEqual } from "node:crypto";
import { eq, isNull } from "drizzle-orm";
import { auth } from "@/features/auth";
import { getDb, apiTokens } from "@/db";
import { ApiError } from "./errors";

export type ApiContext = {
  userId: string;
  tokenId?: string;
};

function sha256(value: string): Buffer {
  return createHash("sha256").update(value).digest();
}

function hashesEqual(a: Buffer, b: Buffer): boolean {
  if (a.length !== b.length) return false;
  return timingSafeEqual(a, b);
}

export async function requireApiAuth(req: Request): Promise<ApiContext> {
  const header = req.headers.get("authorization");
  if (header?.toLowerCase().startsWith("bearer ")) {
    const plaintext = header.slice(7).trim();
    if (!plaintext) throw new ApiError(401, "unauthorized", "Authentication required.");
    const digest = sha256(plaintext);
    const rows = await getDb().query.apiTokens.findMany({
      where: isNull(apiTokens.revokedAt),
      columns: { id: true, userId: true, tokenHash: true },
    });
    const match = rows.find((row) => hashesEqual(Buffer.from(row.tokenHash, "hex"), digest));
    if (!match) throw new ApiError(401, "unauthorized", "Authentication required.");
    return { userId: match.userId, tokenId: match.id };
  }

  const session = await auth();
  if (!session?.user?.id) throw new ApiError(401, "unauthorized", "Authentication required.");
  return { userId: session.user.id };
}

/** UI-only (server components, login). API routes use requireApiAuth. */
export async function requireSession(): Promise<{ userId: string }> {
  const session = await auth();
  if (!session?.user?.id) throw new ApiError(401, "unauthorized", "Authentication required.");
  return { userId: session.user.id };
}
```

Token lookup-by-scan is acceptable for the scaffold (few tokens per user). Do not store the plaintext token. Create endpoint hashes once and returns the secret **once**.

**`src/lib/api/with-api.ts`** — the only `try/catch` on `/api/v1`. Binds request-id logger.
```typescript
import type { NextRequest } from "next/server";
import type { Logger } from "pino";
import { log } from "@/lib/log";
import { applyCors } from "./cors";
import { requireApiAuth, type ApiContext } from "./auth";
import { apiError } from "./response";

export type ApiHandlerContext = ApiContext & { requestId: string; log: Logger; req: NextRequest };

export async function withApi(
  req: NextRequest,
  fn: (ctx: ApiHandlerContext) => Promise<Response>
): Promise<Response> {
  const requestId = (req.headers.get("x-request-id") ?? crypto.randomUUID()).slice(0, 100);
  try {
    const authCtx = await requireApiAuth(req);
    const reqLog = log.child({ requestId, userId: authCtx.userId, tokenId: authCtx.tokenId });
    const res = await fn({ ...authCtx, requestId, log: reqLog, req });
    res.headers.set("x-request-id", requestId);
    res.headers.set("Cache-Control", "private, no-store");
    return applyCors(req, res);
  } catch (err) {
    const res = apiError(err, { requestId });
    res.headers.set("x-request-id", requestId);
    res.headers.set("Cache-Control", "private, no-store");
    return applyCors(req, res);
  }
}
```

**`src/lib/api/client.ts`** — browser + same-origin. Bearer for non-browser: pass `headers: { authorization: \`Bearer ${token}\` }`. `responseSchema` is required so drift fails at the boundary.
```typescript
import { z } from "zod";
import { ApiError } from "./errors";

type RequestOptions<TResponse> = {
  method?: "GET" | "POST" | "PATCH" | "PUT" | "DELETE";
  body?: unknown;
  responseSchema: z.ZodType<TResponse>;
  headers?: Record<string, string>;
  signal?: AbortSignal;
  token?: string;
};

export async function apiFetch<TResponse>(
  path: string,
  options: RequestOptions<TResponse>
): Promise<TResponse> {
  const { method = "GET", body, responseSchema, headers = {}, signal, token } = options;
  const hasBody = body !== undefined;
  const res = await fetch(path, {
    method,
    credentials: "include",
    headers: {
      ...(hasBody ? { "content-type": "application/json" } : {}),
      ...(token ? { authorization: `Bearer ${token}` } : {}),
      ...headers,
    },
    body: hasBody ? JSON.stringify(body) : undefined,
    signal,
  });

  if (res.status === 204) return undefined as TResponse;

  if (!res.ok) {
    const errBody = await res.json().catch(() => ({}));
    throw new ApiError(
      res.status,
      errBody?.error?.code ?? "unknown_error",
      errBody?.error?.message ?? "Request failed"
    );
  }

  const json = await res.json();
  const parsed = responseSchema.safeParse(json?.data ?? json);
  if (!parsed.success) {
    throw new ApiError(500, "invalid_response", "API returned unexpected data shape");
  }
  return parsed.data;
}
```

**`src/lib/log.ts`** — JSON to stdout. Sidecar file is optional. Never throw at import.
```typescript
import pino, { type Logger, type LoggerOptions } from "pino";

const isProd = process.env.NODE_ENV === "production";
const isTest = process.env.NODE_ENV === "test";

const PINO_TO_GCP_SEVERITY: Record<string, string> = {
  trace: "DEBUG", debug: "DEBUG", info: "INFO",
  warn: "WARNING", error: "ERROR", fatal: "CRITICAL",
};

const baseOptions: LoggerOptions = {
  level: process.env.LOG_LEVEL ?? (isProd ? "info" : "debug"),
  serializers: { err: pino.stdSerializers.err, error: pino.stdSerializers.err },
  formatters: {
    level(label) {
      return { level: label, severity: PINO_TO_GCP_SEVERITY[label] ?? label.toUpperCase() };
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

**`src/lib/process-handlers.ts`** — Node-only. Do not put `process.on` in `instrumentation.ts` (Edge scanner).
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
import { getDb, auditLogs } from "@/db";
import { getClientIp } from "@/lib/log";

export type AuditAction =
  | "auth.login"
  | "auth.logout"
  | "user.updated"
  | "user.deleted"
  | "thing.created"
  | "thing.updated"
  | "thing.deleted"
  | "api_token.created"
  | "api_token.revoked";

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

Create `src/db/schema/audit-logs.ts`:
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

Create `src/db/schema/api-tokens.ts` — store **hash only**. Prefix is for UI display (`ns_ab12…`).
```typescript
import { pgTable, text, timestamp, uuid } from "drizzle-orm/pg-core";
import { users } from "./users";

export const apiTokens = pgTable("api_tokens", {
  id: uuid("id").primaryKey().defaultRandom(),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  name: text("name").notNull(),
  tokenHash: text("token_hash").notNull(),
  prefix: text("prefix").notNull(),
  createdAt: timestamp("created_at").notNull().defaultNow(),
  lastUsedAt: timestamp("last_used_at"),
  revokedAt: timestamp("revoked_at"),
});
```

Re-export `apiTokens` from `src/db/schema/index.ts`.

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

Pino is the **only** logger. ESLint `"no-console": "error"` in app code (off in `**/*.test.ts`). See §0 for levels and the split vs HTTP errors.

**Always: context object first, message string second**
```typescript
ctx.log.info({ thingId: row.id }, "things.created");
ctx.log.error({ thingId, err }, "things.create_failed");
```

Event names: `feature.verb` / `feature.verb_state` — `things.created`, `things.delete_failed`, `webhooks.stripe.received`.

On `/api/v1`, use `ctx.log` from `withApi` (already bound to `requestId` + `userId`). Do not call the root `log` from a handler except process-level code.

**Never log secrets** — even at debug. Log shape (`keyPrefix: key.slice(0, 4)`), not the value.

**Client-side errors** — `reportClientError()` + `toast.error`. Never `console.error`, never swallow.

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
      source: source.slice(0, 50),
      url: window.location.href.slice(0, 2_000),
    }),
    keepalive: true,
    credentials: "same-origin",
  }).catch(() => {});
}
```

Create `src/app/api/log/client-error/route.ts` — same-origin, Zod, size cap, IP rate limit. Reconstruct an `Error` so Pino's `err` serializer matches server errors.
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
  route.ts        ← Thin HTTP adapter: withApi → parseBody/parseQuery → handler → apiOk
  [id]/route.ts   ← GET, PATCH, DELETE for a single resource
```

**The split — why `handlers.ts` and `route.ts` are separate files:**

| File | Responsibility | Imports |
|---|---|---|
| `handlers.ts` | Pure business logic. Takes `(ctx, input)`, returns data or throws `ApiError`. No `Request`, no parsing, no auth — `withApi`'s job. | DB queries, audit, log |
| `route.ts` | HTTP adapter. `withApi` auths (cookie or Bearer), `parseBody` / `parseQuery`, calls handler, returns envelope. | `withApi`, schemas, `handlers` |

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

export const ThingPageSchema = z.object({
  items: z.array(ThingSchema),
  next_cursor: z.string().uuid().nullable(),
});

export const ListThingsQuerySchema = z.object({
  cursor: z.string().uuid().optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

export type CreateThingInput = z.infer<typeof CreateThingInputSchema>;
export type UpdateThingInput = z.infer<typeof UpdateThingInputSchema>;
export type Thing = z.infer<typeof ThingSchema>;
export type ThingPage = z.infer<typeof ThingPageSchema>;
export type ListThingsQuery = z.infer<typeof ListThingsQuerySchema>;
```

**`queries.ts`** — DB reads. List is **cursor + stable order** (`created_at desc, id desc`). Never return an unbounded table.
```typescript
import { and, desc, eq, lt, or } from "drizzle-orm";
import { getDb, things } from "@/db";
import type { ListThingsQuery, Thing } from "./schema";

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

export async function listUserThings(userId: string, query: ListThingsQuery): Promise<{ items: Thing[]; next_cursor: string | null }> {
  const db = getDb();
  let cursorRow: ThingRow | undefined;
  if (query.cursor) {
    cursorRow = await db.query.things.findFirst({
      where: and(eq(things.id, query.cursor), eq(things.userId, userId)),
    });
  }
  const rows = await db.query.things.findMany({
    where: cursorRow
      ? and(
          eq(things.userId, userId),
          or(
            lt(things.createdAt, cursorRow.createdAt),
            and(eq(things.createdAt, cursorRow.createdAt), lt(things.id, cursorRow.id))
          )
        )
      : eq(things.userId, userId),
    orderBy: [desc(things.createdAt), desc(things.id)],
    limit: query.limit + 1,
  });
  const extra = rows.length > query.limit;
  const page = extra ? rows.slice(0, query.limit) : rows;
  return {
    items: page.map(toThing),
    next_cursor: extra ? page[page.length - 1]!.id : null,
  };
}

export async function findUserThing(userId: string, id: string): Promise<Thing | null> {
  const row = await getDb().query.things.findFirst({
    where: and(eq(things.id, id), eq(things.userId, userId)),
  });
  return row ? toThing(row) : null;
}
```

**`mutations.ts`** — DB writes (no `"use server"`). Map unique violations to 409.
```typescript
import { and, eq } from "drizzle-orm";
import { getDb, things } from "@/db";
import { ApiError, isPgUniqueViolation } from "@/lib/api/errors";
import type { CreateThingInput, UpdateThingInput, Thing } from "./schema";
import { findUserThing } from "./queries";

export async function insertThing(userId: string, input: CreateThingInput): Promise<Thing | null> {
  try {
    const [row] = await getDb()
      .insert(things)
      .values({ userId, name: input.name, description: input.description ?? null })
      .returning();
    if (!row) return null;
    return findUserThing(userId, row.id);
  } catch (err) {
    if (isPgUniqueViolation(err)) throw new ApiError(409, "conflict", "A thing with those values already exists.");
    throw err;
  }
}

export async function updateThing(
  userId: string,
  id: string,
  fields: UpdateThingInput
): Promise<Thing | null> {
  try {
    await getDb()
      .update(things)
      .set({ ...fields, updatedAt: new Date() })
      .where(and(eq(things.id, id), eq(things.userId, userId)));
    return findUserThing(userId, id);
  } catch (err) {
    if (isPgUniqueViolation(err)) throw new ApiError(409, "conflict", "A thing with those values already exists.");
    throw err;
  }
}

export async function deleteThing(userId: string, id: string): Promise<void> {
  await getDb().delete(things).where(and(eq(things.id, id), eq(things.userId, userId)));
}
```

**`handlers.ts`** — `(ctx, input) → data` or `throw ApiError`. No `Request`, no parse, no auth. Logging uses the root `log` plus `userId` from ctx (requestId is already on the route's child logger when you pass ctx.log — if ctx is only ApiContext, use `log.info({ userId: ctx.userId, ...})`).
```typescript
import type { ApiContext } from "@/lib/api/auth";
import { ApiError } from "@/lib/api/errors";
import { log } from "@/lib/log";
import { logAudit } from "@/lib/audit";
import { findUserThing, listUserThings } from "./queries";
import { insertThing, updateThing, deleteThing } from "./mutations";
import type { CreateThingInput, ListThingsQuery, Thing, ThingPage, UpdateThingInput } from "./schema";

export async function listThings(ctx: ApiContext, query: ListThingsQuery): Promise<ThingPage> {
  return listUserThings(ctx.userId, query);
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

export async function patchThing(ctx: ApiContext, id: string, input: UpdateThingInput): Promise<Thing> {
  if (input.name === undefined && input.description === undefined) {
    throw new ApiError(400, "bad_request", "No fields to update.");
  }
  const existing = await findUserThing(ctx.userId, id);
  if (!existing) throw new ApiError(404, "not_found", "Thing not found.");
  const updated = await updateThing(ctx.userId, id, input);
  if (!updated) throw new ApiError(500, "internal_error", "Could not update thing.");
  log.info({ userId: ctx.userId, thingId: id }, "things.updated");
  await logAudit({ userId: ctx.userId, action: "thing.updated", resource: id });
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

**`src/app/api/v1/things/route.ts`** — thin HTTP adapter. `withApi` is the only catch.
```typescript
import type { NextRequest } from "next/server";
import { withApi } from "@/lib/api/with-api";
import { apiOk } from "@/lib/api/response";
import { parseBody, parseQuery } from "@/lib/api/parse";
import { listThings, createThing } from "@/features/things/handlers";
import { CreateThingInputSchema, ListThingsQuerySchema } from "@/features/things/schema";

export const runtime = "nodejs";

export function GET(req: NextRequest) {
  return withApi(req, async (ctx) => {
    const query = parseQuery(req.url, ListThingsQuerySchema);
    return apiOk(await listThings(ctx, query));
  });
}

export function POST(req: NextRequest) {
  return withApi(req, async (ctx) => {
    const input = await parseBody(req, CreateThingInputSchema);
    return apiOk(await createThing(ctx, input), 201);
  });
}
```

**`src/app/api/v1/things/[id]/route.ts`:**
```typescript
import type { NextRequest } from "next/server";
import { withApi } from "@/lib/api/with-api";
import { apiOk, apiNoContent } from "@/lib/api/response";
import { parseBody, UuidParamSchema } from "@/lib/api/parse";
import { ApiError } from "@/lib/api/errors";
import { getThing, patchThing, removeThing } from "@/features/things/handlers";
import { UpdateThingInputSchema } from "@/features/things/schema";

export const runtime = "nodejs";

type Context = { params: Promise<{ id: string }> };

function parseId(id: string): string {
  const parsed = UuidParamSchema.safeParse(id);
  if (!parsed.success) throw new ApiError(400, "validation_error", "Invalid id");
  return parsed.data;
}

export function GET(req: NextRequest, { params }: Context) {
  return withApi(req, async (ctx) => {
    const { id } = await params;
    return apiOk(await getThing(ctx, parseId(id)));
  });
}

export function PATCH(req: NextRequest, { params }: Context) {
  return withApi(req, async (ctx) => {
    const { id } = await params;
    const input = await parseBody(req, UpdateThingInputSchema);
    return apiOk(await patchThing(ctx, parseId(id), input));
  });
}

export function DELETE(req: NextRequest, { params }: Context) {
  return withApi(req, async (ctx) => {
    const { id } = await params;
    await removeThing(ctx, parseId(id));
    return apiNoContent();
  });
}
```

**`api.ts`** — client-side fetch wrappers (used by React Query hooks):
```typescript
import { z } from "zod";
import { apiFetch } from "@/lib/api/client";
import {
  ThingSchema,
  ThingPageSchema,
  type CreateThingInput,
  type UpdateThingInput,
  type Thing,
  type ThingPage,
  type ListThingsQuery,
} from "./schema";

export function listThings(query: ListThingsQuery = { limit: 20 }): Promise<ThingPage> {
  const q = new URLSearchParams();
  if (query.cursor) q.set("cursor", query.cursor);
  q.set("limit", String(query.limit ?? 20));
  return apiFetch(`/api/v1/things?${q}`, { responseSchema: ThingPageSchema });
}

export function getThing(id: string): Promise<Thing> {
  return apiFetch(`/api/v1/things/${id}`, { responseSchema: ThingSchema });
}

export function createThing(input: CreateThingInput): Promise<Thing> {
  return apiFetch("/api/v1/things", { method: "POST", body: input, responseSchema: ThingSchema });
}

export function updateThing(id: string, input: UpdateThingInput): Promise<Thing> {
  return apiFetch(`/api/v1/things/${id}`, { method: "PATCH", body: input, responseSchema: ThingSchema });
}

export function deleteThing(id: string): Promise<void> {
  return apiFetch(`/api/v1/things/${id}`, { method: "DELETE", responseSchema: z.undefined() });
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

**Success (resource):**
```json
{ "data": { "id": "...", "name": "..." } }
```

**Success (list):**
```json
{ "data": { "items": [], "next_cursor": null } }
```

**Error (any 4xx or 5xx):**
```json
{ "error": { "code": "validation_error", "message": "Name is required" } }
```

Echo `x-request-id` on every `/api/v1` response. 204 DELETE has no body.

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
      auth.ts              ← requireApiAuth, ApiContext
      with-api.ts          ← withApi
      parse.ts             ← parseBody, parseQuery
      cors.ts
      client.ts            ← apiFetch
    env.ts                 ← lazy getters; never throw at import
    app-environment.ts     ← APP_ENV local|dev|prod
    log.ts
    process-handlers.ts    ← uncaughtException / unhandledRejection
    audit.ts
    get-ip.ts
    rate-limit.ts          ← IP sliding window; Redis for this only
    webhook.ts             ← HMAC on raw body
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
      api-tokens.ts
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
import { sql } from "drizzle-orm";
import { getDb } from "@/db";

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
APP_ENV=local
LOG_LEVEL=debug
DD_SERVICE={{APP_NAME}}
# APP_LOG_FILE=/shared-volume/logs/app.log

# CORS allowlist for /api/v1 (comma-separated). Same-origin UI does not need this.
CORS_ORIGINS=http://localhost:3000

# Webhook HMAC (raw-body verify). Optional until you add a vendor.
WEBHOOK_SECRET=

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

**ESLint — `eslint.config.mjs`.** Keep Next's defaults and set:
```js
{
  rules: { "no-console": "error" },
}
```
In the `ignores` / file override for `**/*.{test,spec}.{ts,tsx}` and `scripts/**`, set `"no-console": "off"`.

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

Write `STANDARDS.md` in full from this template. Do not stub, summarize, or omit rules. It is the contract for humans and coding agents and must match §0.

```markdown
# STANDARDS.md

Engineering rules for this repo. Every code change should satisfy them.

## Architecture (keep it modular)

- Feature folders: `schema → queries → mutations → handlers → api → hooks → index → components`
- Cross-feature imports go through `index.ts`. Never deep-import another feature's internals.
- `handlers.ts` is HTTP-agnostic — never imports `Request`, `NextResponse`, or `headers()`. Pure `(ctx, input) → output`.
- `route.ts` is a thin adapter: `withApi` → `parseBody` / `parseQuery` → handler → `apiOk` / `apiNoContent`. No business logic, no second `try/catch`.
- Queries read. Mutations write. Handlers orchestrate. Routes do HTTP. Do not collapse these files.
- `"use server"` only in `features/auth/actions.ts`. Every other mutation goes through `/api/v1/*`.
- `page.tsx` / `layout.tsx` never use `"use client"`. Push interactivity into a child.
- Server components import `queries.ts` directly. Client components use React Query hooks wrapping `api.ts`.

## Errors, logging, observability (four layers — do not mix)

1. **Exceptions:** handlers throw `ApiError` for expected 4xx/429, or a real `Error` for crashes. `withApi` is the only `try/catch` on `/api/v1`.
2. **HTTP errors:** envelope only — `{ data }` / `{ error: { code, message } }`. Never put `err.message` or a stack in the body. Other users' ids → 404, not 403.
3. **Logs:** Pino only. `log.<level>({ ...context }, "feature.verb")`. No `console.*`. Use `ctx.log` on `/api/v1` (already bound to `requestId` + `userId`).
4. **Audit:** `log.info` is not the audit trail. Writes also call `logAudit`. GETs do neither.

- `fatal` = process going down · `error` = this request failed · `warn` = recoverable (expected 4xx, Redis down) · `info` = state change · `debug` = off in prod
- Honor or mint `x-request-id`; echo it; bind `log.child`. Dual fields: `level` + GCP `severity`.
- IP is personal data. Store it on `audit_logs` and on `client.error_reported` / `request.error`. Do **not** bind IP on every `log.info`.
- `/api/health` does `SELECT 1` and returns 503 if the DB is down. Probe this, not `/`.
- Client errors POST to `/api/log/client-error` (same-origin, Zod, size cap, IP cap).

## API

1. No `any`, no `as` on user input — Zod at every boundary. `safeParse` + `firstError`. Never `.parse()`.
2. Every `/api/v1` route goes through `withApi` — cookie or Bearer via `requireApiAuth`. Never skip it.
3. Envelope + status table: 201 create, 204 delete, 404 other users' ids, 409 unique (`23505`), 413 body cap, 415 content-type, 429 + `Retry-After`.
4. Lists are cursor-paginated (`limit` 1–100). Never unbounded `findMany()`.
5. `apiFetch({ responseSchema })` is required, even for our own API.
6. Routes live under `/api/v1/`. Additive fields on v1 are ok; breaking changes go to `/api/v2/`.
7. `parseBody` / `parseQuery` — never `req.json()` on `/api/v1`.
8. CORS allowlist only (`CORS_ORIGINS`). Never `*`.
9. Authenticated responses set `Cache-Control: private, no-store`. Do not CDN-cache or Redis-cache `/api/v1` GET bodies. React Query (`staleTime` + `invalidateQueries` on write) is the client cache. Redis is for **rate limiting only**.

## Auth, secrets, webhooks, rate limit

- API tokens: store SHA-256 hash + display prefix. Compare with `timingSafeEqual`. Never store plaintext.
- Webhooks: HMAC on the **raw body**, then parse. Idempotent. Not rate-limited (signature is the gate).
- Rate limit `/api/v1` by IP in middleware (60/min sliding window). Fail **open** if Redis is missing or the limiter throws. Key is IP, never user id alone. `"unknown"` IPs share one bucket.
- No top-level `throw` for runtime env vars — lazy getter (`getDb()`, `databaseUrl()`). `next build` must import the module.
- No secret in code, committed env files, or `NEXT_PUBLIC_*`.

## UI, tests, git

- Component filenames are kebab-case. No hardcoded color / px — tokens + Tailwind scale. Use shadcn; never duplicate.
- Every form: `react-hook-form` + `zodResolver` + `useMutation`.
- Static assets in `public/`, `next/image`, `next/font`.
- Test every handler: happy path + validation-fail + not-found. Assert log event names on writes.
- `APP_ENV` drives source maps (on local/dev, off prod). Not `NODE_ENV`.
- Commit messages: `type: description` (`feat|fix|docs|style|refactor|test|chore|perf`).
- Comments explain the code's why, not ticket numbers or rollout sequencing.
```

---

### 19. CLAUDE.md

Write `CLAUDE.md` in full. Do not stub. It is the orientation file for coding agents.

```markdown
# CLAUDE.md

Auto-loaded by Claude Code. Companion files (read before changing code):
- [`AGENTS.md`](./AGENTS.md) — router + non-negotiables
- [`STANDARDS.md`](./STANDARDS.md) — engineering rules
- [`REVIEW.md`](./REVIEW.md) — review rubric

Never commit, push, branch, or open a PR unless the user explicitly asks. Never pass `--no-verify`.

---

## Repo orientation

Next.js 16 app with **strict REST APIs**. Bun = package manager, Node.js = runtime. The only `"use server"` file is NextAuth sign-in/sign-out.

| Surface | Where |
|---|---|
| Authenticated UI | `src/app/(platform)/` |
| Public pages | `src/app/(auth)/`, `src/app/terms`, `src/app/privacy` |
| Versioned REST API | `src/app/api/v1/<resource>/route.ts` |
| Webhooks | `src/app/api/webhooks/<vendor>/route.ts` |
| Health | `src/app/api/health/route.ts` |
| Client error ingest | `src/app/api/log/client-error/route.ts` |
| Business logic | `src/features/<name>/handlers.ts` |
| DB reads / writes | `src/features/<name>/queries.ts` / `mutations.ts` |
| Shared HTTP + log | `src/lib/api/*`, `src/lib/log.ts`, `src/lib/audit.ts` |
| DB schema | `src/db/schema/` |

---

## Canonical feature (keep this split)

        src/features/<name>/
          schema.ts       ← Zod request + response
          queries.ts      ← DB reads (SSR + handlers)
          mutations.ts    ← DB writes (handlers only)
          handlers.ts     ← (ctx, input) → output; no HTTP
          api.ts          ← browser fetch wrappers
          hooks/          ← React Query
          index.ts        ← public barrel
          components/

`route.ts` = `withApi` (auth + one catch + CORS + request id + `Cache-Control: private, no-store`) → `parseBody`/`parseQuery` → handler → `apiOk`.
`handlers.ts` never imports `Request`, `NextResponse`, or `next/headers`.
Cross-feature imports go through `index.ts`.

---

## Four layers (do not mix)

        throw ApiError / Error  →  withApi catch  →  envelope (user) + log (operator)
        log.info / warn / error →  stdout JSON. Operators only.
        logAudit(...)           →  audit_logs. Durable who-did-what.

- Expected failure = `ApiError`. Crash = `Error`. Never leak `err.message` into the HTTP body.
- On `/api/v1` use `ctx.log`, already bound to `requestId` + `userId`. Signature: `log.<level>({ ...ctx, err }, "feature.verb")`.
- IP belongs on `audit_logs` and on `client.error_reported` / `request.error`. Not on every `log.info`.
- Redis is the **rate limiter**, not a response cache. Client cache is React Query (`staleTime` + `invalidateQueries`). Authenticated GET bodies are `private, no-store`.

---

## How to... (playbooks)

### Add a new feature

Run `bun run create-feature <kebab-name>`. Then: DB table, `AuditAction`, `/api/v1/<name>/` routes via `withApi`, components.

### Add a new API endpoint
1. Schemas in `feature/schema.ts`
2. Handler in `feature/handlers.ts` — `(ctx, input)`, throw `ApiError` or return data. `log.info` + `logAudit` on writes.
3. `src/app/api/v1/<resource>/route.ts` — `withApi` + `parseBody`/`parseQuery` only
4. Typed wrapper in `feature/api.ts` with `responseSchema`
5. React Query hook; mutations `invalidateQueries` on success
6. Tests: happy path + validation-fail + not-found; assert log event names on writes

### Add a server action
**Don't.** Only `features/auth/actions.ts` for sign-in/sign-out. Everything else is `/api/v1/*`.

### Add a DB column or table
1. Edit `src/db/schema/<table>.ts`
2. `bunx drizzle-kit generate` — commit SQL **and** `drizzle/meta/`
3. `bunx drizzle-kit migrate` locally
Adding `NOT NULL` to a populated table without a default fails on deploy. Nullable → backfill → tighten.

### Add an env var
1. Lazy getter in `src/lib/env.ts` — never throw at import
2. `.env.example`
3. `instrumentation.ts` warn-list if required at runtime
4. CI `env:` block
Runtime secrets are not `NEXT_PUBLIC_*`. `APP_ENV` is a **build ARG** (`local|dev|prod`); `NODE_ENV` does not drive source maps.

### Add a webhook handler
1. `src/app/api/webhooks/<vendor>/route.ts` with `runtime = "nodejs"`
2. HMAC on **raw body** (`req.text()`), then parse, then Zod
3. Idempotent upsert on event id
4. Do not rate-limit — signature is the gate

### Rate limit a specific route
Throw `ApiError(429, "rate_limited", "...", retryAfter)` **inside** `withApi`. Do not add a second catch. Default blunt limit is already IP 60/min on all `/api/v1` in middleware; fail open if Redis is down.

---

## Gotchas

- **`page.tsx` is never `"use client"`.** Extract a child.
- **SSR reads `queries.ts`, not the HTTP API.**
- **Additive fields on `/api/v1` are ok. Breaking changes go to `/api/v2/`.** Do not silently change a published request/response shape.
- **Envelope is mandatory.** `apiOk` / `apiError` / `apiNoContent` only.
- **`getDb()` is lazy.** A top-level `new Pool(process.env.DATABASE_URL)` dies in `next build`.
- **Do not cache `/api/v1` in Redis or a CDN.** User-scoped JSON will leak across users.

---

## Commands

| Command | What |
|---|---|
| `bun install` | Install deps |
| `bun run dev` | Dev server :3000 |
| `bun run build` | Production build (`APP_ENV` must be set on the image) |
| `bun run create-feature <name>` | Scaffold a feature |
| `bunx vitest run` | Tests |
| `bunx tsc --noEmit` | Typecheck |
| `bunx next lint` | ESLint |
| `bunx drizzle-kit generate` | Generate migration |
| `bunx drizzle-kit migrate` | Apply migrations |
| `bun audit --audit-level=high` | Security audit |
```

---

### 20. AGENTS.md

Write `AGENTS.md` in full. It is the **router**: non-negotiables that apply to every change, plus a map. Detailed rules live in `STANDARDS.md` and `REVIEW.md`.

```markdown
# AGENTS.md

## Coding principles

- YAGNI. Prefer the existing helper (`withApi`, `parseBody`, `log`, `logAudit`) over a new abstraction.
- Comments explain the code's why, not ticket numbers or rollout sequencing.

## Work management

- Never commit, push, branch, or run `gh pr create` / `gh pr merge` unless the user explicitly asks. Never `--no-verify`.
- Never automatically reply to human review comments on a PR.

**This file is a router.** Read the file named for the task before you write the change.

| Before you… | Read |
|---|---|
| Write a route, handler, query, schema, or feature folder | [`STANDARDS.md`](./STANDARDS.md) |
| Review a diff / self-review before a PR | [`REVIEW.md`](./REVIEW.md) |
| Orient in the repo / add a feature | [`CLAUDE.md`](./CLAUDE.md) |

---

## Non-negotiables

1. Every `/api/v1` route goes through `withApi`. That is the only `try/catch`. `requireApiAuth` (cookie or Bearer) runs inside it. Never skip it.
2. Handlers throw `ApiError` for expected failures. Unknown errors become `500 { error: { code: "internal_error", message: "Something went wrong." } }`. Never put `err.message` or a stack in the HTTP body.
3. `handlers.ts` does not import `Request`, `NextResponse`, or `next/headers`.
4. `"use server"` only in `features/auth/actions.ts`. Every other mutation is `/api/v1/*`.
5. Pino only. `log.<level>({ ...context }, "feature.verb")`. No `console.*`. Writes also call `logAudit`. GETs do neither.
6. Webhooks verify HMAC against the **raw body** before parse. Idempotent.
7. No secret in code, committed env, or `NEXT_PUBLIC_*`. No module-top-level `throw` on a runtime env var — lazy getter (`getDb()`).
8. Rate limit `/api/v1` by IP in middleware; fail open if Redis is missing. Redis is **not** a response cache. Authenticated JSON is `Cache-Control: private, no-store`.
9. IP is personal data: `audit_logs` + client-error / `onRequestError` only. Not on every log line.
10. Other users' ids → **404**, not 403. Lists are cursor-paginated. CORS allowlist, never `*`.
11. `APP_ENV` (build ARG) drives source maps. `NODE_ENV` does not.
12. Feature folders stay split: `schema → queries → mutations → handlers → api → hooks → index → components`. Cross-feature via `index.ts`.

---

## Review guidelines

### 🔴 Block-merge (always raise)

- **`"use server"` outside `features/auth/actions.ts`**
- **Auth bypass** — `/api/v1` routes that skip `withApi` / `requireApiAuth`
- **Second `try/catch` in a route** — stacks leak or get double-logged; `withApi` is the only catch
- **Webhook `req.json()` before HMAC** — must `req.text()`, verify, then parse
- **Plaintext secrets** — token columns without SHA-256 + `timingSafeEqual`
- **Module-load throws for runtime env vars** — breaks `next build`
- **`NOT NULL` on a populated table without a default**
- **Hardcoded secrets / committed `.env`**
- **Envelope bypassed** — not using `apiOk` / `apiError` / `apiNoContent`
- **`handlers.ts` importing HTTP types**
- **`"use client"` on `page.tsx` / `layout.tsx`**
- **Redis or CDN cache of authenticated `/api/v1` GET bodies** — cross-user leak
- **`Cache-Control: public` on `/api/v1`**
- **`console.*` in app code** (tests/scripts exempt)

### 🟡 Discuss / suggest

- Missing `logAudit` on a write (reads don't need it)
- New `AuditAction` not added to the union
- Missing `invalidateQueries` on mutation success
- Wrong HTTP status (200 instead of 201/204)
- IP bound onto every `log.info` — keep it on audit + abuse paths
- Rate limiter keyed by user id alone
- Hardcoded color / px; PascalCase component filenames
- `useState` + `useEffect` fetching; forms without `react-hook-form` + `zodResolver`

### ⚪ Skip / don't comment

- Style nits Prettier fixes
- Theoretical races without an attack path
- Missing tests for trivial helpers
- DoS / resource exhaustion under realistic load
- Outdated transitive deps
- "Add OpenTelemetry / Sentry / Redis response cache" unless the app already has a sink

## Files to skip entirely

| Path | Reason |
|---|---|
| `**/drizzle/0*.sql`, `**/drizzle/meta/**` | Generated |
| `bun.lock`, `package-lock.json`, `yarn.lock` | Lockfiles |
| `**/*.test.ts`, `**/*.test.tsx` | Reviewed by humans |
| `**/_template/**` | Scaffolding source |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Context, not code |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |
| `**/public/images/**` | Static assets |

## Confidence calibration

- Read the file, not just the diff
- Don't suggest renames or refactors that aren't behavior changes
- Don't recommend a component that doesn't exist
- Match existing patterns
- If unsure, don't flag unless 🔴 or 🟡
```

---

### 21. REVIEW.md

Write `REVIEW.md` in full. Bots and humans use this file. Keep it in sync with `AGENTS.md` and `STANDARDS.md`.

```markdown
# REVIEW.md

Code-review rubric. Applies to humans and bots.

Read [`CLAUDE.md`](./CLAUDE.md) and [`STANDARDS.md`](./STANDARDS.md) first. `AGENTS.md` is the router.

---

## Severity rubric

### 🔴 Block-merge

- **`"use server"` outside `features/auth/actions.ts`**
- **Auth bypass** — missing `withApi` / `requireApiAuth`
- **Second `try/catch` in a `/api/v1` route** — `withApi` is the only catch
- **Webhook signature missing or `req.json()` before verify**
- **Module-load throws for env vars** — breaks `next build`
- **`NOT NULL` on a populated table without default**
- **FK violations** — non-user UUIDs in `audit_logs.user_id`
- **`apiOk` / `apiError` / `apiNoContent` bypassed**
- **`handlers.ts` importing HTTP types**
- **`"use client"` on `page.tsx` / `layout.tsx`**
- **Plaintext token/secret storage**
- **Redis/CDN cache of authenticated `/api/v1` GET bodies**, or `Cache-Control: public` on `/api/v1`
- **`err.message` / stack in an HTTP error body**
- **`console.*` in app code**

### 🟡 Discuss

- Missing `invalidateQueries` on mutation success
- Missing `logAudit` on a write
- New `AuditAction` not added to the union
- Wrong HTTP status (200 for create; body on 204)
- IP on every log line (keep it on audit + abuse paths)
- Rate limit keyed by user id alone; fail-closed when Redis is unset
- Hardcoded color/px; PascalCase filenames
- `useState` + `useEffect` fetching; form without `react-hook-form` + `zodResolver`
- Native `confirm()` / `alert()` — use shadcn `Dialog`
- Duplicate shadcn components — extend via `cva`

### ⚪ Skip

- Auto-fixable style nits
- Theoretical race conditions
- Missing tests for trivial helpers
- DoS / resource exhaustion
- Outdated transitive deps
- "Add Sentry / OpenTelemetry / Redis response cache" with no sink in this repo

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

- **Breaking a published request/response shape** — additive fields on v1 are ok; breaking changes go to `/api/v2/`
- **Removing an endpoint** — deprecate first
- **Wrong status** — POST create = 201, DELETE = 204, other users' ids = 404
- **Skipping `withApi` / `parseBody`**
- **Returning internals** (`err.message`, stack) in the envelope
- **Missing `x-request-id` or `Cache-Control: private, no-store`**

### 🟡 Discuss

- Adding an optional response field — fine, document
- Adding a new endpoint under an existing resource — fine
- Loosening request validation
- Per-route tighter rate limit (throw `ApiError(429)` inside `withApi`)

---

## Logging / observability PR rubric

### 🔴 Block-merge

- `console.*` in app code
- Missing `{ err }` on a failure log (loses stack)
- Logging `authorization`, cookies, tokens, or plaintext secrets

### 🟡 Discuss

- Event name not `feature.verb`
- Write without `log.info` + `logAudit`
- IP added to the default `withApi` child logger

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
- [ ] If touching `/api/v1/*`: went through `withApi`; envelope unchanged for existing endpoints; writes have `log.info` + `logAudit`
- [ ] If adding a webhook: HMAC on raw body, then parse; idempotent
- [ ] No secrets in code / `NEXT_PUBLIC_*`; no module-top-level env throws

## Test plan

<!-- How to verify this works -->

-
```

---

### 23. `next.config.ts`

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

### 24. `instrumentation.ts`

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

**`src/lib/rate-limit.ts`** — lazy Redis. Missing Upstash → fail **open** + `log.warn`. Never `Redis.fromEnv()` at import (`next build` would die).
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
  return async function check(ip: string): Promise<RateLimitResult> {
    const redis = getRedis();
    if (!redis) {
      log.warn({}, "ratelimit.disabled_no_redis");
      return { allowed: true };
    }
    const limiter = new Ratelimit({
      redis,
      limiter: Ratelimit.slidingWindow(config.max, `${config.windowMs}ms`),
      prefix: `rl:${config.windowMs}:${config.max}`,
    });
    const { success, reset } = await limiter.limit(ip);
    if (!success) return { allowed: false, retryAfter: Math.ceil((reset - Date.now()) / 1000) };
    return { allowed: true };
  };
}
```

Per-route tighter limit — throw `ApiError(429, "rate_limited", "...", retryAfter)` **inside** `withApi` so `Retry-After` is set. Do not add a second try/catch.

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

### 26b. Webhooks — raw body, then parse

Canonical handler. Copy this; swap the header name for Stripe/GitHub/Slack. **Never** `req.json()` before verify.

**`src/lib/webhook.ts`:**
```typescript
import { createHmac, timingSafeEqual } from "node:crypto";
import { ApiError } from "@/lib/api/errors";
import { webhookSecret } from "@/lib/env";

export function verifyHmacSha256(rawBody: string, header: string | null): void {
  const secret = webhookSecret();
  if (!secret) throw new ApiError(500, "internal_error", "Webhook is not configured.");
  if (!header) throw new ApiError(401, "unauthorized", "Missing signature.");
  const digest = createHmac("sha256", secret).update(rawBody).digest("hex");
  const a = Buffer.from(digest);
  const b = Buffer.from(header);
  if (a.length !== b.length || !timingSafeEqual(a, b)) {
    throw new ApiError(401, "unauthorized", "Invalid signature.");
  }
}
```

**`src/app/api/webhooks/stripe/route.ts`** (generic HMAC; rename header when you wire a vendor):
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
  try {
    verifyHmacSha256(raw, req.headers.get("x-webhook-signature"));
  } catch (err) {
    log.warn({ err }, "webhooks.signature_rejected");
    return NextResponse.json({ error: { code: "unauthorized", message: "Invalid signature." } }, { status: 401 });
  }
  let json: unknown;
  try {
    json = JSON.parse(raw);
  } catch {
    return NextResponse.json({ error: { code: "bad_request", message: "Invalid payload." } }, { status: 400 });
  }
  const parsed = EventSchema.safeParse(json);
  if (!parsed.success) {
    return NextResponse.json({ error: { code: "bad_request", message: "Invalid payload." } }, { status: 400 });
  }
  // Idempotent: upsert on parsed.data.id, then return 200 even on retries.
  log.info({ eventId: parsed.data.id, type: parsed.data.type }, "webhooks.received");
  return NextResponse.json({ received: true });
}
```

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

All should pass before the first commit. `next build` must succeed with dummy `DATABASE_URL` (lazy `getDb()`). Confirm `productionBrowserSourceMaps` is false when `APP_ENV=prod`.

**Post-scaffold (human, not the model):** Neon project, Google OAuth client, Upstash Redis, Vercel tokens, optional tweakcn theme, `WEBHOOK_SECRET` when you add a vendor.

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
| shadcn/ui | CSS variables, no hardcoded colors (tweakcn optional later) |
| Drizzle + Postgres | Lazy `getDb()`, TLS verify on, pooled URL for app, direct URL for migrations |
| NextAuth v5 + Google | JWT sessions; adapter only for users/accounts. Sign-in/out is the only `"use server"` |
| **REST `/api/v1` + `withApi`** | Cookie **or** Bearer. One catch. Envelope + request id + CORS + `private, no-store` |
| **Cursor pagination** | `{ items, next_cursor }`, limit 1–100 |
| **Webhooks** | HMAC on **raw body**, then parse; idempotent |
| Pino logger | JSON + GCP severity, `APP_LOG_FILE` sidecar, redaction, `no-console` |
| Client errors | Same-origin, Zod, rate-limited `/api/log/client-error` |
| Source maps | `APP_ENV` build ARG — on local/dev, **off prod** |
| Audit trail | `log.info` + `logAudit` on every write including PATCH. IP on audit, not every log line |
| Rate limit | IP middleware; Redis for this only; fail open if Redis missing. No Redis response cache |
| Client cache | React Query `staleTime` + `invalidateQueries` on write |
| Governance docs | `CLAUDE.md` (orient) · `AGENTS.md` (router) · `STANDARDS.md` (rules) · `REVIEW.md` (rubric) — written in full |
| Dockerfile | Bun→Node 22, `APP_ENV` + `NEXT_PUBLIC_*` only at build — no secrets in the image |
| `apiFetch` client | Typed fetch + required `responseSchema`; cookie or Bearer |
| Zod at every boundary | One schema, two consumers — `safeParse` on server, `zodResolver` on client |
| Forms | `react-hook-form` + `zodResolver` + `useMutation` |
| Feature template | `schema → queries → mutations → handlers → api → hooks → index → components` |
| Vitest | Handler + hook + schema tests — 3-case minimum |
| `instrumentation.ts` | Env at first use + `onRequestError` |
| Error/loading pages | `error.tsx`, `global-error.tsx`, `not-found.tsx`, `loading.tsx` |
| Governance docs | `CLAUDE.md`, `STANDARDS.md`, `AGENTS.md`, `REVIEW.md` |
| CI | Parallel lint / format / typecheck / audit / test |

## PROMPT END
