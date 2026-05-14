# Next.js Production Scaffold Prompt

Copy-paste this entire prompt to scaffold a new production-ready Next.js app from scratch.
Replace `{{APP_NAME}}` with your app name (kebab-case, e.g. `my-app`).

---

## PROMPT START

Scaffold a production-ready Next.js app called `{{APP_NAME}}`. Follow every instruction exactly — don't add extras, don't skip steps.

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

Do not hand-write CSS variables. tweakcn generates the full `:root` + `.dark` variable set that shadcn components expect (`--background`, `--foreground`, `--primary`, `--muted`, `--border`, `--ring`, etc.).

**Rules for using shadcn:**
- Always install a component via `bunx shadcn@latest add <name>` — never copy-paste component code manually
- Never duplicate a shadcn component — if `Button` exists, use it everywhere, don't make a `CustomButton`
- Compose complex UI by combining shadcn primitives — don't rewrite them
- If you need a variant that doesn't exist, extend the existing component's `variants` config using `cva`, don't create a parallel component

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
  session: { strategy: "jwt" },
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

**`src/features/auth/index.ts`** — full NextAuth with Drizzle adapter:
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

  // Rate limit public API routes by IP — skip auth routes (NextAuth handles
  // those) and skip webhook endpoints (signature verification is their gate).
  const isPublicApi = pathname.startsWith("/api/") && !pathname.startsWith("/api/auth");
  if (isPublicApi) {
    const result = await apiLimiter(getIp(req));
    if (!result.allowed) {
      return NextResponse.json(
        { error: "Too many requests." },
        { status: 429, headers: { "Retry-After": String(result.retryAfter) } }
      );
    }
  }

  // Auth guard — redirect unauthenticated users to login
  const isLoggedIn = !!req.auth;
  const isAuthRoute = pathname.startsWith("/api/auth");
  const isPublicPage = ["/login", "/terms", "/privacy"].includes(pathname);

  if (isAuthRoute || isPublicPage) return;
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

**`src/lib/audit.ts`:**
```typescript
import { headers } from "next/headers";
import { db } from "@/db";
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
export function reportClientError(err: unknown, context?: Record<string, unknown>): void {
  const message = err instanceof Error ? err.message : String(err);
  const stack = err instanceof Error ? err.stack : undefined;
  fetch("/api/log/client-error", {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify({ message, stack, context }),
  }).catch(() => {}); // best-effort
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
  } catch {
    // malformed body — ignore
  }
  return NextResponse.json({ ok: true });
}
```

---

### 11. Feature folder pattern

Create `src/features/_template/` with these files. This is the canonical shape every new feature copies.

**`schema.ts`** — Zod input validation only. Every string field gets `.max()` with a custom message. Every error message is user-facing.

**`queries.ts`** — Read-only Drizzle queries. Export typed row types.

**`actions.ts`** — Server actions (`"use server"` at the top of the file, never inline). Every action follows this exact order:
1. `auth()` → check `session?.user?.id`
2. `Schema.safeParse(input)` → return `fail(firstError(...))`
3. DB write
4. `log.info({ userId, ...context }, "feature.verb")`
5. `await logAudit({ userId, action: "feature.verb", resource: id })`
6. `revalidatePath(...)` on affected paths
7. `return ok(result)`

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
    log.ts
    audit.ts
    get-ip.ts
    rate-limit.ts
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
LOG_LEVEL=debug

# Upstash Redis (rate limiting)
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

Create `STANDARDS.md` with these rules (one section per rule):

1. **No `any`, no `as` casts on user input** — validate with Zod at every boundary
2. **Every server action returns `ActionResult<T>`** — no thrown errors crossing the boundary
3. **`safeParse` in actions, `firstError` for the user message** — never `.parse()`
4. **Auth gate is the first line of every action** — before any DB call
5. **No `console.log` server-side** — use `log` from `@/lib/log`
6. **`log.info` + `logAudit` after every write** — state changes have an audit trail
7. **`revalidatePath` after every mutation** — on the narrowest affected set of paths
8. **No top-level `throw` for runtime env vars** — lazy getter pattern only
9. **`"use server"` only at the top of files, never inline** — extract inline server actions into a `actions.ts` file
10. **Commit messages: `type: description`** — enforced by commit-msg hook
11. **Test every action: happy path + auth-fail + validation-fail** — minimum 3 cases
12. **Feature folders: `schema → queries → actions → index → components`** — no exceptions
13. **Cross-feature imports go through `index.ts`** — never deep-import into another feature
14. **Component filenames are kebab-case** — `home-navbar.tsx`, not `HomeNavbar.tsx`
15. **No hardcoded color values** — use semantic tokens only (`bg-background`, `text-muted-foreground`, `border-border`)
16. **No hardcoded `px` values** — use Tailwind spacing scale; arbitrary values only when the scale has no match
17. **Use shadcn components, never duplicate them** — extend via `cva` variants, don't create parallel components
18. **Rate limit every unauthenticated public route by IP** — never user ID alone; webhooks are exempt (signature verification is their gate)
19. **`page.tsx` files never use `"use client"`** — pages stay server components so SSR'd content reaches the crawler. If a page needs interactivity, extract the interactive parts into a child component file marked `"use client"`, and have the page render it
20. **Static assets live in `public/`, referenced with absolute paths** — `/logo.png` not `./logo.png`. No secrets, no private docs, no files >1MB in `public/`
21. **Use `next/image` for raster images and `next/font` for fonts** — never `<img>` or self-hosted font files unless there's a documented reason
22. **Every form uses `react-hook-form` + `zodResolver`** — the same Zod schema the server action validates with is the form's resolver. No hand-rolled `useState` form state, no parallel validation logic

---

### 18. CLAUDE.md

Create `CLAUDE.md` at repo root — auto-loaded by Claude Code, also read by other AI tools:

```markdown
# CLAUDE.md

Auto-loaded by Claude Code at the start of every session. Companion files:

- [`STANDARDS.md`](./STANDARDS.md) — the engineering rules; every code change should satisfy them
- [`REVIEW.md`](./REVIEW.md) — review rubric (🔴 / 🟡 / ⚪ severity)
- [`AGENTS.md`](./AGENTS.md) — entry point for AI review bots (Codex, Devin, Gemini)

---

## Never run git commands — show them instead

Never execute `git add`, `git commit`, `git push`, `git merge`, `git rebase`, or any other mutating git command. Read-only inspection (`git status`, `git diff`, `git log`) is fine.

Paste the exact commands the user should run themselves. Same rule for `gh pr create`, `gh pr merge`, and anything that mutates branches or PRs.

**Why:** the user reviews staged changes before they hit history. Auto-committing moves work into the remote without the user inspecting the diff.

---

## Repo orientation

Single Next.js 16 app. Bun = package manager + script runner. Node.js = app runtime.

| Surface | Where it lives |
|---|---|
| Authenticated UI | `src/app/(platform)/` |
| Public pages (login, terms, privacy) | `src/app/(auth)/`, `src/app/terms`, etc. |
| Public API | `src/app/api/v1/` |
| Webhooks | `src/app/api/webhooks/` |
| Business logic | `src/features/<name>/` |
| Shared utilities | `src/lib/` |
| DB schema | `src/db/schema/` |

---

## Canonical feature reference

When in doubt about how a feature should look, read `src/features/_template/`. Every other feature follows the same shape:

```
src/features/<name>/
  schema.ts       ← Zod input validation, exported types
  queries.ts      ← Drizzle reads, typed row types
  actions.ts      ← "use server" at top; auth → safeParse → write → log → audit → revalidate → ok()
  index.ts        ← public barrel: page components + actions + types
  components/     ← client + server components for this feature only
```

**Cross-feature imports go through `index.ts`** — never deep-import past a feature's barrel. The exception is sub-path imports needed to keep server-only modules out of client/edge bundles (see "Gotchas" below).

---

## How to... (playbooks)

### Add a new feature
```bash
bun run create-feature <kebab-name>
```
Then follow the printed steps: add the DB table, extend `AuditAction`, uncomment skeleton imports, build components, add the route.

### Add a server action
1. Add input schema to `feature/schema.ts` with `.max()` + custom messages on every string
2. Write the action: `auth() → safeParse → write → log.info → logAudit → revalidatePath → ok(...)`
3. Re-export from `feature/index.ts`
4. Test: happy path + auth-fail + validation-fail

### Add a DB column or table
1. Edit `src/db/schema/<table>.ts`
2. Run `bunx drizzle-kit generate` — commit both the SQL and `drizzle/meta/`
3. `bunx drizzle-kit migrate` to apply locally

⚠️ Adding `NOT NULL` to a populated table without a default fails on deploy. Make it nullable, backfill, then tighten.

### Add an env var
1. Read it via a lazy memoized getter — never top-level `throw` on missing env (breaks `next build`)
2. Add to `.env.example`
3. Add to the `REQUIRED` list in `src/instrumentation.ts`
4. Add to CI workflow's `env:` block

### Add a webhook handler
1. Route at `src/app/api/webhooks/<vendor>/route.ts` with `export const runtime = "nodejs"`
2. Verify signature against the **raw body** before parsing JSON
3. Make the handler idempotent (webhooks retry)
4. Skip rate limiting for webhook routes — signature verification is the gate

### Add a public API route
1. Authenticate via `authenticateApiToken(req)` — never trust headers alone
2. Rate limit per IP with `createRateLimiter({...})`
3. Validate body with Zod `safeParse`
4. Return a typed JSON response with appropriate HTTP status

---

## Gotchas (non-obvious)

**`page.tsx` files never use `"use client"`.** Adding `"use client"` to a route entry kills server rendering — the crawler sees an empty shell instead of your content, breaking SEO and link previews. If a page needs interactivity, extract it into a child component (e.g. `_components/add-to-cart-button.tsx`) and import it. Same rule for `layout.tsx` and any other route-entry file.

**Sub-path imports for client/edge code.** A feature's `index.ts` re-exports page components, which are server components that transitively import server-only code (`pg`, etc.). If a **client component** or `middleware.ts` imports from `@/features/X`, the edge bundler walks the whole barrel and errors with `Module not found: Can't resolve 'child_process'`. Fix: import from the sub-path (`@/features/X/lib/something`), not the barrel.

**`"use server"` files cannot re-export types.** Next.js's RPC scanner treats every export as a server action. Re-export types from a non-`"use server"` module.

**No top-level throws for runtime env vars.** `next build`'s "Collecting page data" step imports every route module in Node. If a module top-levels `if (!process.env.X) throw ...`, the build dies in CI where X isn't set. Use a lazy memoized getter.

**`audit_logs.user_id` is FK→`users.id`, nullable.** System-triggered audit entries (webhooks, crons) pass `userId: null`, not some other UUID. The FK rejects anything else.

**Bearer token parsing is case-insensitive.** RFC 7235. Datadog / PagerDuty / Grafana send `bearer ` (lowercase). Use `.toLowerCase().startsWith("bearer ")`.

---

## Commands

| Command | What |
|---|---|
| `bun install` | Install deps (frozen-lockfile in CI) |
| `bun run dev` | Start dev server on :3000 |
| `bun run build` | Production build |
| `bun run create-feature <name>` | Scaffold a new feature from the template |
| `bunx vitest run` | Run all tests |
| `bunx tsc --noEmit` | Typecheck |
| `bunx next lint` | ESLint |
| `bunx drizzle-kit generate` | Generate migration from schema diff |
| `bunx drizzle-kit migrate` | Apply migrations |
| `bunx drizzle-kit studio` | DB GUI on :4983 |
| `bun audit --audit-level=high` | Security audit |
```

---

### 19. AGENTS.md

Create `AGENTS.md` at repo root — entry point for AI review bots (Codex, Devin, Gemini Code Assist):

```markdown
# AGENTS.md

Configuration for AI code-review agents (Codex, Gemini Code Assist, Claude Code, Devin) working in this repository.

This file is the entry point. The full rubric and codebase context live in three companion files; read them in order before reviewing:

1. **[`CLAUDE.md`](./CLAUDE.md)** — repo orientation, gotchas, "how to add X" playbooks, and conventions
2. **[`STANDARDS.md`](./STANDARDS.md)** — engineering rules that apply to every change
3. **[`REVIEW.md`](./REVIEW.md)** — full review rubric with severity tiers

What follows is a high-leverage summary so you don't review blind even if you haven't loaded the full files.

---

## Review guidelines

### 🔴 Block-merge (always raise)

- **Auth bypass** — public route handlers that skip authentication, middleware bypass-list growth without justification, or NextAuth callbacks that trust unvalidated JWT claims
- **Webhook signature missing** — Stripe / Slack / GitHub webhooks must verify signatures **before** dispatching to handlers
- **Plaintext secret storage** — any new secret column without SHA-256 hashing + `timingSafeEqual` comparison
- **Module-load throws for runtime env vars** — breaks `next build`'s page-data collection. Use lazy memoized getters
- **`NOT NULL` added to a populated table without a default** — fails on deploy. Split into nullable → backfill → tighten
- **FK violations** — passing non-user UUIDs to `audit_logs.user_id` (FK→`users.id`)
- **Hardcoded secrets in code or committed env files** — must be in GCP Secret Manager / equivalent, never in repo
- **Missing rate limiting on unauthenticated public routes** — every public route needs a limiter; webhooks are exempt (signature is their gate)

### 🟡 Discuss / suggest

- Missing `revalidatePath` after a mutation
- Missing `logAudit` on a state-changing action (reads don't need audit; mutations do)
- New `AuditAction` value used without adding it to the union in `src/lib/audit.ts`
- Action returning a non-`ActionResult<T>` shape
- `"use server"` declared inline rather than at the top of an `actions.ts` file
- Hardcoded color values (`text-red-500`, `bg-[#1a1a1a]`) — use semantic tokens
- Hardcoded `px` values (`p-[14px]`, `w-[320px]`) — use Tailwind scale
- Component filename in PascalCase — should be kebab-case
- `"use client"` on a `page.tsx` or `layout.tsx` — kills SSR/SEO; extract interactivity into a child component
- Hand-rolled `useState` form state — every form must use `react-hook-form` + `zodResolver(Schema)` with the same Zod schema the server action validates
- Top-level barrel imports in client/edge code — Turbopack will misclassify; use sub-path imports

### ⚪ Skip / don't comment

- Style-only nits already auto-fixed by ESLint or Prettier in pre-commit
- Theoretical race conditions without a concrete attack path
- Missing tests for trivial helpers
- DoS / resource exhaustion concerns under realistic load
- Outdated transitive dependencies (handled separately)

---

## Files to skip entirely

Don't post comments on these paths — they're generated, lockfiles, or meta-context rather than reviewable code:

| Path | Reason |
|---|---|
| `**/drizzle/0*.sql`, `**/drizzle/meta/**` | Generated by `drizzle-kit`. Review `src/db/schema/`, not the SQL |
| `bun.lock`, `package-lock.json`, `yarn.lock` | Lockfiles |
| `**/*.test.ts`, `**/*.test.tsx` | Tests are reviewed by humans; bots have poor signal here |
| `**/_template/**` | Scaffolding source |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Reviewer reads these for context |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |
| `**/public/images/**` | Static images |

---

## Confidence calibration

This codebase ships fast. Before posting a comment:

- **Read the file you're commenting on**, not just the diff snippet. Many flags are wrong because they rely on the local diff and miss a guard or import elsewhere in the file.
- **Don't suggest renames or refactors** that aren't behavior changes unless they're explicitly in the rules above.
- **Don't recommend a component that doesn't exist** — verify it's in `@/components/ui/` before suggesting it.
- **Match existing patterns** — if the codebase uses `!==` for a token comparison everywhere, don't suggest `timingSafeEqual` for that one surface unless the secret is stored hashed.
- If unsure between flagging and not, **don't flag** unless the issue is in the 🔴 or 🟡 lists.

---

## Per-PR overrides

For one-off review focus, leave a comment on the PR like:

- `@codex review for security regressions`
- `@codex review with extra scrutiny on /api/v1/**`
- `@gemini-code-assist focus on the billing feature`

These override the defaults for that PR only.
```

---

### 20. REVIEW.md

Create `REVIEW.md` at repo root — full code-review rubric for humans and bots:

```markdown
# REVIEW.md

Code-review guidelines for this repository. Apply them whether you're reviewing as a human or as an AI bot — the calibration is the same.

For codebase context, read [`CLAUDE.md`](./CLAUDE.md) and [`STANDARDS.md`](./STANDARDS.md) first. The rules below are the **review rubric**.

---

## Severity rubric

### 🔴 Block-merge (always raise)

- **Auth bypass** — public route handlers that skip authentication; middleware bypass-list growth without a self-auth justification; NextAuth callbacks that trust unvalidated JWT claims
- **Webhook signature missing** — Stripe / Slack / GitHub webhooks must verify signatures **before** dispatching to handlers
- **Plaintext secret storage** — any new secret column without SHA-256 hashing + `timingSafeEqual` comparison
- **Module-load throws for runtime env vars** — breaks `next build`'s page-data collection in CI. Use lazy memoized getters
- **`NOT NULL` added to a populated table without a default** — fails on deploy. Split into nullable → backfill → tighten
- **FK / NOT NULL violations** — passing non-user UUIDs to `audit_logs.user_id`, or `undefined` to required columns via partial `set()`
- **Untrusted metadata reaching enum / index sinks** — webhook body fields cast to typed enums without Zod validation
- **Missing rate limiting on unauthenticated public routes** — webhooks exempt (signature verification is their gate)

### 🟡 Discuss / suggest

- Missing `revalidatePath` after a mutation
- Missing `logAudit` on a state-changing action
- New `AuditAction` value used without adding it to the union in `src/lib/audit.ts`
- Action returning a non-`ActionResult<T>` shape
- `"use server"` declared inline inside JSX rather than at the top of an `actions.ts` file
- Hardcoded color values (`text-red-500`, `bg-[#1a1a1a]`) — use semantic tokens (`text-destructive`, `bg-card`)
- Hardcoded `px` values (`p-[14px]`, `w-[320px]`) — use Tailwind scale (`p-3.5`, `w-80`)
- Component filename in PascalCase — should be kebab-case
- `"use client"` on a `page.tsx` or `layout.tsx` — kills SSR/SEO; extract interactivity into a child component
- Hand-rolled `useState` form state — every form must use `react-hook-form` + `zodResolver(Schema)` with the same Zod schema the server action validates (`home-navbar.tsx`, not `HomeNavbar.tsx`)
- Top-level barrel imports in client/edge code — Turbopack will misclassify
- Native `confirm()` / `alert()` in admin UI — use the shadcn `Dialog` primitive
- Duplicate shadcn components — extend via `cva` variants, don't create parallel components

### ⚪ Skip / don't comment

- Style-only nits already auto-fixed by ESLint or Prettier in pre-commit
- Theoretical race conditions without a concrete attack path
- Missing tests for trivial helpers — the project doesn't aim for 100% coverage
- Lack of audit logs on non-state-changing reads
- DoS / resource exhaustion concerns under realistic load
- Outdated transitive dependencies (handled separately)

### Files to skip entirely

| Path | Reason |
|---|---|
| `**/drizzle/0*.sql` | Generated by `drizzle-kit`. Review `src/db/schema/`, not the SQL |
| `**/drizzle/meta/**` | Generated snapshot files |
| `bun.lock`, `package-lock.json`, `yarn.lock` | Lockfiles |
| `**/*.test.ts`, `**/*.test.tsx` | Tests are reviewed by humans |
| `**/_template/**` | Scaffolding source |
| `STANDARDS.md`, `CLAUDE.md`, `REVIEW.md`, `AGENTS.md` | Reviewer reads these for context — doesn't review them as code |
| `**/.next/**`, `**/dist/**`, `**/build/**` | Build artifacts |
| `**/public/images/**` | Static images |

When the diff is _primarily_ in these files (e.g., a "regenerate lockfile" PR), post a single approving comment instead of going line-by-line.

---

## Migration / schema PR rubric

When a PR has changes under `drizzle/` or `src/db/schema/`, run this extra checklist before any per-line comments. Migration PRs are high-blast-radius.

### 🔴 Block-merge

- **Schema and SQL drift** — `src/db/schema/<table>.ts` doesn't match the generated `drizzle/000N_*.sql`. Either the SQL was hand-edited or `drizzle-kit generate` wasn't re-run
- **`NOT NULL` added to a populated table without a default** — will fail in prod on deploy
- **FK to a table that doesn't exist yet** in the migration order — Postgres rejects the migration
- **Dropping a column / table referenced by live code** — grep `src/` for the column name; if it's still read or written, the deploy breaks the moment the migration runs
- **Renaming a column without a data-preserving plan** — Drizzle generates `ADD` + `DROP` for renames, which loses data. Either keep the old column for a release, dual-write, then drop in a follow-up

### 🟡 Discuss

- **New enum value added to the postgres enum but not the TypeScript literal type** (or vice versa) — they have to ship together
- **Index on a column without a query that needs it** — index bloat
- **`drizzle-kit generate` produced statements you didn't expect** — e.g., a migration touches an unrelated column because the schema file shifted

### ⚪ Skip

- The exact SQL formatting in `drizzle/000N_*.sql` — generated
- The `meta/000N_snapshot.json` contents — generated

### Things to verify with the actual code, not the PR description

- "Backfill is safe" → grep for callers of the affected table
- "This rename is on an empty table" → check the table really is empty in current production
- "We added a default" → confirm the default is in the generated SQL, not just the TS schema

The PR description is intent; the diff is reality. When they conflict, raise it.

---

## Confidence calibration

- **Read the file you're commenting on**, not just the diff snippet
- **Don't suggest renames or refactors** that aren't behavior changes unless they're in the rules above
- **Don't recommend a component that doesn't exist** — verify it's installed first
- **Match existing patterns** rather than imposing external conventions
- If unsure between flagging and not, **don't flag** unless the issue is in the 🔴 or 🟡 lists
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

## Test plan

<!-- How to verify this works -->

-
```

---

### 22. `next.config.ts`

Create `next.config.ts` at the app root:

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // Required for Docker / Cloud Run deploys — builds a self-contained
  // output directory instead of relying on node_modules at runtime.
  output: "standalone",

  // Native-Node packages that pull in `fs`, `child_process`, `dns` via
  // dynamic require — Turbopack/webpack can't trace them statically.
  // Symptom when missing: "Module not found: Can't resolve 'child_process'"
  // during `next build`. Add the top-level package whose transitive tree fails.
  serverExternalPackages: [
    "pg",
    "pino",
  ],

  // ESM-only packages that need to be transpiled for Next.js.
  transpilePackages: [],

  images: {
    remotePatterns: [
      // Google OAuth profile pictures
      { protocol: "https", hostname: "lh3.googleusercontent.com" },
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

### 23. `instrumentation.ts`

Create `src/instrumentation.ts`:

```typescript
/**
 * Next.js server instrumentation — runs once on startup.
 * Validates required env vars and wires up global error capture.
 */
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

  // Fail fast on missing required env vars. Log loudly so a bad deploy
  // surfaces immediately rather than silently breaking mid-request.
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

  const appUrl = process.env.NEXT_PUBLIC_APP_URL;
  if (appUrl) {
    try {
      const parsed = new URL(appUrl);
      if (process.env.NODE_ENV === "production" && parsed.protocol !== "https:") {
        log.error({ appUrl }, "instrumentation.app_url_not_https");
      }
      if (appUrl.endsWith("/")) {
        log.warn({ appUrl }, "instrumentation.app_url_trailing_slash");
      }
    } catch {
      log.error({ appUrl }, "instrumentation.app_url_invalid");
    }
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

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    console.error(error);
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
    // Prefix isolates each limiter's keys in Redis — prevents collisions
    // when two routes share the same window + max config.
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
- Always use IP as the key — never user ID alone (logged-out users have no ID)
- Apply to every **unauthenticated** public route — if anyone can call it without a session, it needs a limiter
- **Do not** rate limit webhook endpoints (Stripe, Slack, GitHub) — use signature verification; their IPs rotate and a limit will drop legitimate events
- **Do not** rate limit authenticated routes — use per-account quota (billing layer) instead
- Always return `Retry-After` header — clients and browsers respect it
- `"unknown"` IPs share one bucket — this is intentional; it's a safe fallback that still throttles

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

# Build-time env vars — pass via --build-arg or your platform's build settings.
# Required because `next build` collects page data at build time.
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

# ── Stage 3: Production runtime (Node, not Bun — Next.js runtime stays Node) ─
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

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
*.lock
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
| Bun package manager | Fast installs, Vitest integration. Next.js runtime stays on Node.js |
| Husky hooks | Enforces format, lint, types, tests, audit on every push |
| Conventional commits | Enforced by commit-msg hook |
| Feature template | Every feature has the same shape — onboard anyone instantly |
| `bun run create-feature` | One-command scaffolder copies template, replaces tokens, prints next steps |
| `ActionResult<T>` | No thrown errors crossing server/client boundary |
| Zod conventions | `safeParse` only, every string `.max()`, user-facing messages, boundary-only validation |
| Forms | `react-hook-form` + `zodResolver` everywhere — same schema as the server action, no `useState` form state |
| Pino logger | Structured JSON, GCP-severity compatible, secret redaction, `feature.verb` event names |
| Client error reporting | `/api/log/client-error` + `reportClientError()` — no swallowed errors |
| Audit trail | Every write has `logAudit` — compliance-ready from day one |
| Drizzle + NeonDB | Schema-first migrations, pooled URL for app, direct URL for drizzle-kit |
| Upstash Redis | Serverless Redis for rate limiting — edge + Node.js compatible |
| NextAuth v5 + Google | Production auth with extracted server actions, no inline `"use server"` |
| Vitest + mock builders | Consistent 3-case test pattern across all features |
| shadcn/ui + tweakcn theme | CSS variables from tweakcn.com, full token set, no hardcoded colours |
| Styling conventions | Kebab-case filenames, no hardcoded px/colours, Tailwind scale + semantic tokens |
| `.env.example` | Every env var documented, nothing secret in git |
| `next.config.ts` | `standalone` output, `serverExternalPackages`, security headers |
| `instrumentation.ts` | Startup env validation + global `onRequestError` capture with digest |
| Error/loading pages | `error.tsx`, `global-error.tsx`, `not-found.tsx`, `loading.tsx` |
| Rate limiting | IP-based, Upstash sliding window, works in middleware + route handlers |
| CI workflow | Parallel lint/format/typecheck/audit/test jobs, build gates on all passing |
| Governance docs | `CLAUDE.md`, `STANDARDS.md`, `AGENTS.md`, `REVIEW.md` — full rule + review rubric |
| PR template | `.github/pull_request_template.md` — checklist for every change |
| Email | Nodemailer + Gmail SMTP via App Password — swap transport for Resend/Postmark later |
| SEO | `sitemap.ts`, `robots.ts`, OG image conventions, root + per-page metadata patterns |
| Vercel deploy | GitHub Actions workflow + `vercel.json` to disable auto-deploy |
| Dockerfile | Multi-stage Bun→Node build using `output: "standalone"` for non-Vercel hosts |

## PROMPT END
