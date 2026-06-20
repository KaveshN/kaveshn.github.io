---
date: 2026-05-06
authors:
  - kavesh
categories:
  - Identity
  - Infrastructure
tags:
  - Entra ID
  - OAuth
  - Next.js
slug: entra-nextjs-getting-started
description: >-
  A complete, self-contained walkthrough from zero to a working
  Entra-authenticated Next.js app on your laptop — prerequisites, localhost,
  and Microsoft sign-in end to end.
---

# Skills Directory — Getting Started Guide

A complete, self-contained walkthrough for getting from zero to a working Entra-authenticated Next.js app on your laptop. Covers Day 0 (prerequisites), Day 1 (project on localhost), and Day 2 (Microsoft sign-in working end-to-end).

This is the on-ramp to Sprint 1 of the project roadmap. By the end, you'll have a real, working foundation that everything else builds on.

<!-- more -->

---

## Day 0 — Prerequisites (2–3 hours)

### Install on your machine

| Tool | Why | How |
|---|---|---|
| **Node.js 22 LTS** (or 25 Current) | Runtime | nvm recommended: `nvm install 22 && nvm use 22`. Node 25 also works for dev — just set `"engines"` in package.json to target 22 for production parity with Vercel. |
| **pnpm** | Package manager (faster than npm, better monorepo support) | `npm install -g pnpm` |
| **Git** | Version control | Likely already installed |
| **VS Code** (or Cursor) | Editor | With extensions: ESLint, Prettier, Prisma, Tailwind CSS IntelliSense |
| **GitHub CLI (`gh`)** | Repo creation, PR management | `brew install gh` (Mac) or equivalent |
| **Docker Desktop** *(optional)* | Local Postgres if you don't want to use Neon for dev | Skip if using Neon dev branches |
| **Azure CLI** | Entra app registration via CLI (faster than the portal) | `brew install azure-cli` |

Verify with:
```bash
node --version    # v22.x or v25.x both fine
pnpm --version
git --version
gh --version
az --version
```

### Create accounts (all free tiers)

1. **GitHub** — create a private repo named `skills-directory` (don't initialise it; you'll push from local).
2. **Vercel** — sign up with your GitHub account. Free Hobby tier is fine for now.
3. **Neon** — sign up with GitHub. Free tier gives you 1 project with branching. Create a project called `skills-directory` in **EU Central** region.
4. **Sentry** — sign up, create a project of type "Next.js". Save the DSN.
5. **Microsoft Entra dev tenant** — confirm you can sign in to `entra.microsoft.com`.

That's the entire stack you need for Phase 1, Sprint 1. Nothing paid yet.

### Get your bearings in your Entra dev tenant

Before registering the app, log in to `entra.microsoft.com` and confirm:

- **Your tenant ID** — Overview → copy the Tenant ID (a UUID). You'll need this.
- **You have admin rights** — Roles & admins → confirm Global Administrator or Application Administrator.
- **You have at least 2–3 test users besides yourself** — Users → if it's just you, create test users with mock job titles, departments, and photos. The directory needs *something* to render, and a "directory of one" is hard to demo.

> **Tip:** put fake but realistic data on those test users — different job titles, different departments. When you build the search/filter UI, real-looking data makes the UX easier to evaluate.

---

## Day 1 — Get the app running on localhost (4–6 hours)

This is the "I have something on `localhost:3000`" milestone. No Entra yet, no DB queries — just the skeleton.

### Step 1: Initialise the repo

```bash
mkdir skills-directory && cd skills-directory
git init
gh repo create skills-directory --private --source=. --remote=origin
```

### Step 2: Set up the monorepo

```bash
pnpm init
# Edit package.json — add "private": true and the workspace config below
```

Create `pnpm-workspace.yaml`:
```yaml
packages:
  - "apps/*"
  - "packages/*"
```

Create the directory structure:
```bash
mkdir -p apps/web packages/shared packages/config infra docs/adr docs/compliance docs/runbooks
```

### Step 3: Scaffold the Next.js app

```bash
cd apps/web
pnpm create next-app@latest . --typescript --tailwind --app --src-dir --import-alias "@/*" --eslint --no-turbopack
```

When prompted, accept defaults. This gives you Next.js 15 with App Router, TypeScript strict, Tailwind, and ESLint pre-configured.

### Step 4: Pin Node version for production parity

Edit `apps/web/package.json` and add:
```json
{
  "engines": {
    "node": ">=22 <23"
  }
}
```

This tells Vercel to use Node 22 LTS in production, regardless of what version you develop on locally.

### Step 5: Add shadcn/ui

```bash
# Still in apps/web/
pnpm dlx shadcn@latest init
```

Choose: New York style, Slate base color, CSS variables yes. Then add base components:

```bash
pnpm dlx shadcn@latest add button card input label avatar badge dialog dropdown-menu sonner
```

### Step 6: Install core dependencies

```bash
pnpm add @azure/msal-node @azure/msal-react @azure/msal-browser @microsoft/microsoft-graph-client
pnpm add @prisma/client zod react-hook-form @hookform/resolvers
pnpm add @tanstack/react-query iron-session jose
pnpm add @sentry/nextjs pino
pnpm add -D prisma vitest @playwright/test @types/node
```

### Step 7: Set up Prisma with two schemas

The control plane and tenant data plane each need their own Prisma schema.

```bash
pnpm prisma init
mkdir -p prisma/control prisma/tenant
mv prisma/schema.prisma prisma/control/schema.prisma
```

Create `apps/web/prisma/control/schema.prisma`:
```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../../node_modules/.prisma/control"
}

datasource db {
  provider = "postgresql"
  url      = env("CONTROL_DATABASE_URL")
}

model Tenant {
  id              String   @id @default(uuid())
  displayName     String
  entraTenantId   String   @unique
  regionCode      String   @default("eu-west-1")
  status          String   @default("active") // active | suspended | offboarding
  dpaSignedAt     DateTime?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  @@map("tenants")
}

model Region {
  code                  String   @id // e.g. "eu-west-1", "za-north-1"
  displayName           String
  connectionStringRef   String   // name of the Vercel env var, not the value
  isDefault             Boolean  @default(false)
  createdAt             DateTime @default(now())

  @@map("regions")
}
```

Create `apps/web/prisma/tenant/schema.prisma`:
```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../../node_modules/.prisma/tenant"
}

datasource db {
  provider = "postgresql"
  url      = env("TENANT_DATABASE_URL")
}

model User {
  id              String   @id @default(uuid())
  tenantId        String
  externalId      String   // Entra OID
  provider        String   @default("entra")
  email           String
  displayName     String
  jobTitle        String?
  department      String?
  officeLocation  String?
  photoUrl        String?
  photoHash       String?
  disabledAt      DateTime?
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt

  userSkills      UserSkill[]
  certifications  Certification[]

  @@unique([tenantId, externalId, provider])
  @@index([tenantId])
  @@map("users")
}

model Skill {
  id           String   @id @default(uuid())
  tenantId     String
  name         String
  category     String?
  isCurated    Boolean  @default(false)
  createdAt    DateTime @default(now())

  userSkills   UserSkill[]

  @@unique([tenantId, name])
  @@index([tenantId])
  @@map("skills")
}

model UserSkill {
  id           String   @id @default(uuid())
  tenantId     String
  userId       String
  skillId      String
  proficiency  String   // "Beginner" | "Intermediate" | "Advanced" | "Expert"
  createdAt    DateTime @default(now())

  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  skill        Skill    @relation(fields: [skillId], references: [id], onDelete: Cascade)

  @@unique([tenantId, userId, skillId])
  @@index([tenantId])
  @@map("user_skills")
}

model Certification {
  id            String   @id @default(uuid())
  tenantId      String
  userId        String
  name          String
  issuer        String
  issuedAt      DateTime?
  expiresAt     DateTime?
  credentialUrl String?
  createdAt     DateTime @default(now())

  user          User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([tenantId])
  @@map("certifications")
}

model AuditLog {
  id              String   @id @default(uuid())
  tenantId        String
  actorUserId     String?
  actorExternalId String?
  action          String   // e.g. "user.signin", "skill.add", "profile.view"
  resourceType    String?
  resourceId      String?
  metadata        Json?
  ipAddress       String?
  userAgent       String?
  createdAt       DateTime @default(now())

  @@index([tenantId, createdAt])
  @@index([tenantId, actorUserId])
  @@map("audit_log")
}
```

### Step 8: Wire up environment variables

Create `apps/web/.env.local` (gitignored):

```bash
# Control plane DB (your Neon project)
CONTROL_DATABASE_URL="postgresql://...@ep-xxx.eu-central-1.aws.neon.tech/skills_control?sslmode=require"

# Default tenant region DB
TENANT_DATABASE_URL_EU_WEST_1="postgresql://...@ep-xxx.eu-central-1.aws.neon.tech/skills_tenant_eu?sslmode=require"

# Entra (you'll fill these in on Day 2)
ENTRA_CLIENT_ID=""
ENTRA_CLIENT_SECRET=""
ENTRA_TENANT_ID="common"
ENTRA_REDIRECT_URI="http://localhost:3000/api/auth/callback"

# Session
SESSION_SECRET="generate_a_64_char_random_string_here"

# Sentry
NEXT_PUBLIC_SENTRY_DSN=""
SENTRY_AUTH_TOKEN=""
```

Generate the session secret:
```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

Also create `apps/web/.env.example` with the same keys but no values — commit this so future-you (or a collaborator) knows what's needed.

### Step 9: Create your two databases on Neon

In the Neon console:
1. Your existing project (let's call it `skills-directory`) already has a default database. Rename it `skills_control`, or create a new one with that name.
2. Add a second database in the same project: `skills_tenant_eu`.
3. Copy both connection strings into `.env.local`.

### Step 10: Run your first migrations

```bash
# From apps/web/
pnpm prisma migrate dev --schema=./prisma/control/schema.prisma --name init
pnpm prisma migrate dev --schema=./prisma/tenant/schema.prisma --name init
```

If both succeed, you have two databases with empty tables. **This is a milestone — celebrate it.**

### Step 11: Run the dev server

```bash
pnpm dev
```

Open `http://localhost:3000`. You'll see the default Next.js landing page.

**Day 1 done.** You have a running Next.js app, two databases, and your project structure. No auth yet, but the foundations are real.

---

## Day 2 — Get Entra sign-in working (4–6 hours)

This is the moment of truth — when "I built a Next.js app" becomes "I built a Next.js app that signs in with Microsoft."

### Step 1: Register your app in Entra

The CLI way (faster):

```bash
az login --tenant <YOUR_DEV_TENANT_ID>

az ad app create \
  --display-name "Skills Directory (dev)" \
  --sign-in-audience AzureADMultipleOrgs \
  --web-redirect-uris "http://localhost:3000/api/auth/callback" \
  --enable-id-token-issuance true \
  --required-resource-accesses '[
    {
      "resourceAppId": "00000003-0000-0000-c000-000000000000",
      "resourceAccess": [
        {"id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d", "type": "Scope"},
        {"id": "b340eb25-3456-403f-be2f-af7a0d370277", "type": "Scope"}
      ]
    }
  ]'
```

The two scopes are `User.Read` and `User.ReadBasic.All` (delegated).

This returns an `appId` — that's your `ENTRA_CLIENT_ID`. Save it.

Create a client secret:
```bash
az ad app credential reset --id <YOUR_APP_ID> --display-name "dev-secret" --years 1
```

This returns a `password` — that's your `ENTRA_CLIENT_SECRET`. **You only see this once — copy it immediately into `.env.local`.**

> **Portal alternative:** `entra.microsoft.com → App registrations → New registration`. Name it "Skills Directory (dev)", choose "Accounts in any organisational directory", redirect URI `http://localhost:3000/api/auth/callback`, register. Then API permissions → Add → Microsoft Graph → Delegated → `User.Read`, `User.ReadBasic.All`. Then Certificates & secrets → New client secret.

### Step 2: Build the auth abstraction (the strategy interface)

Create `apps/web/src/lib/auth/identity-provider.ts`:

```typescript
export interface NormalizedIdentity {
  externalId: string;
  provider: 'entra' | 'workos' | 'google' | 'okta';
  entraTenantId: string;
  email: string;
  emailVerified: boolean;
  displayName: string;
  rawClaims: Record<string, unknown>;
}

export interface IdentityProvider {
  getAuthorizationUrl(state: string, nonce: string): Promise<string>;
  handleCallback(code: string, expectedNonce: string): Promise<NormalizedIdentity>;
  signOut(postLogoutRedirectUri: string): Promise<string>;
}
```

Create `apps/web/src/lib/auth/providers/msal-provider.ts`:

```typescript
import { ConfidentialClientApplication, type Configuration } from '@azure/msal-node';
import { jwtVerify, createRemoteJWKSet } from 'jose';
import type { IdentityProvider, NormalizedIdentity } from '../identity-provider';

const config: Configuration = {
  auth: {
    clientId: process.env.ENTRA_CLIENT_ID!,
    clientSecret: process.env.ENTRA_CLIENT_SECRET!,
    authority: `https://login.microsoftonline.com/${process.env.ENTRA_TENANT_ID ?? 'common'}`,
  },
};

const msalClient = new ConfidentialClientApplication(config);

const SCOPES = ['openid', 'profile', 'email', 'User.Read', 'User.ReadBasic.All'];
const REDIRECT_URI = process.env.ENTRA_REDIRECT_URI!;

const JWKS = createRemoteJWKSet(
  new URL('https://login.microsoftonline.com/common/discovery/v2.0/keys'),
);

export const msalProvider: IdentityProvider = {
  async getAuthorizationUrl(state, nonce) {
    return msalClient.getAuthCodeUrl({
      scopes: SCOPES,
      redirectUri: REDIRECT_URI,
      state,
      nonce,
      responseMode: 'query',
    });
  },

  async handleCallback(code, expectedNonce) {
    const result = await msalClient.acquireTokenByCode({
      code,
      scopes: SCOPES,
      redirectUri: REDIRECT_URI,
    });

    if (!result?.idToken) {
      throw new Error('No ID token returned from Entra');
    }

    const { payload } = await jwtVerify(result.idToken, JWKS, {
      audience: process.env.ENTRA_CLIENT_ID,
    });

    if (payload.nonce !== expectedNonce) {
      throw new Error('Nonce mismatch');
    }

    return {
      externalId: payload.oid as string,
      provider: 'entra',
      entraTenantId: payload.tid as string,
      email: (payload.preferred_username ?? payload.email) as string,
      emailVerified: true,
      displayName: payload.name as string,
      rawClaims: payload as Record<string, unknown>,
    };
  },

  async signOut(postLogoutRedirectUri) {
    return `https://login.microsoftonline.com/common/oauth2/v2.0/logout?post_logout_redirect_uri=${encodeURIComponent(postLogoutRedirectUri)}`;
  },
};
```

### Step 3: Build the session helpers

Create `apps/web/src/lib/auth/session.ts`:

```typescript
import { getIronSession, type IronSession, type SessionOptions } from 'iron-session';
import { cookies } from 'next/headers';

export interface AppSession {
  userId?: string;
  tenantId?: string;
  externalId?: string;
  email?: string;
  displayName?: string;
  authNonce?: string; // used during auth flow only
  authState?: string; // used during auth flow only
}

const sessionOptions: SessionOptions = {
  password: process.env.SESSION_SECRET!,
  cookieName: 'skills_directory_session',
  cookieOptions: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    sameSite: 'lax',
    maxAge: 60 * 60 * 8, // 8 hours
  },
};

export async function getSession(): Promise<IronSession<AppSession>> {
  return getIronSession<AppSession>(await cookies(), sessionOptions);
}
```

### Step 4: Sign-in route

Create `apps/web/src/app/api/auth/signin/route.ts`:

```typescript
import { NextResponse } from 'next/server';
import { randomBytes } from 'node:crypto';
import { msalProvider } from '@/lib/auth/providers/msal-provider';
import { getSession } from '@/lib/auth/session';

export async function GET() {
  const state = randomBytes(16).toString('hex');
  const nonce = randomBytes(16).toString('hex');

  const session = await getSession();
  session.authState = state;
  session.authNonce = nonce;
  await session.save();

  const url = await msalProvider.getAuthorizationUrl(state, nonce);
  return NextResponse.redirect(url);
}
```

### Step 5: Callback route

Create `apps/web/src/app/api/auth/callback/route.ts`:

```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { msalProvider } from '@/lib/auth/providers/msal-provider';
import { getSession } from '@/lib/auth/session';

export async function GET(req: NextRequest) {
  const url = new URL(req.url);
  const code = url.searchParams.get('code');
  const state = url.searchParams.get('state');

  if (!code || !state) {
    return NextResponse.redirect(new URL('/?error=missing_code', req.url));
  }

  const session = await getSession();

  if (state !== session.authState || !session.authNonce) {
    return NextResponse.redirect(new URL('/?error=invalid_state', req.url));
  }

  try {
    const identity = await msalProvider.handleCallback(code, session.authNonce);

    // For Day 2, we just store the identity — no DB upsert yet.
    // Sprint 2 adds tenant resolution + user upsert.
    session.externalId = identity.externalId;
    session.email = identity.email;
    session.displayName = identity.displayName;
    session.authState = undefined;
    session.authNonce = undefined;
    await session.save();

    return NextResponse.redirect(new URL('/me', req.url));
  } catch (err) {
    console.error('Auth callback failed', err);
    return NextResponse.redirect(new URL('/?error=auth_failed', req.url));
  }
}
```

### Step 6: A "you're signed in" page

Create `apps/web/src/app/me/page.tsx`:

```typescript
import { redirect } from 'next/navigation';
import { getSession } from '@/lib/auth/session';

export default async function MePage() {
  const session = await getSession();

  if (!session.externalId) {
    redirect('/');
  }

  return (
    <div className="mx-auto max-w-2xl p-8">
      <h1 className="text-2xl font-semibold">Signed in</h1>
      <dl className="mt-6 grid grid-cols-2 gap-4 text-sm">
        <dt className="font-medium">Display name</dt>
        <dd>{session.displayName}</dd>
        <dt className="font-medium">Email</dt>
        <dd>{session.email}</dd>
        <dt className="font-medium">External ID</dt>
        <dd className="font-mono text-xs">{session.externalId}</dd>
      </dl>
    </div>
  );
}
```

### Step 7: Update the home page with a sign-in button

Replace `apps/web/src/app/page.tsx`:

```typescript
import Link from 'next/link';
import { Button } from '@/components/ui/button';

export default function Home() {
  return (
    <main className="flex min-h-screen flex-col items-center justify-center gap-6">
      <h1 className="text-4xl font-bold">Skills Directory</h1>
      <p className="text-muted-foreground">Sign in with your Microsoft account.</p>
      <Button asChild>
        <Link href="/api/auth/signin">Sign in with Microsoft</Link>
      </Button>
    </main>
  );
}
```

### Step 8: The moment of truth

```bash
pnpm dev
```

Open `http://localhost:3000`. Click sign in. You should be redirected to Microsoft, sign in with one of your dev tenant's test users, and bounce back to `/me` showing your name, email, and OID.

**If it works: you've crossed the hardest single mountain in this whole build.** Entra OIDC working end-to-end is the gnarly bit. From here on, everything else is "build features on top."

### Common Day 2 failure modes

- **"AADSTS50011: redirect URI mismatch"** — your `.env.local` redirect URI doesn't exactly match what's registered in Entra. They must match character-for-character including the trailing slash (or lack thereof).
- **"invalid_state"** — your cookie isn't persisting. Check that you're on `http://localhost:3000` not `127.0.0.1:3000`.
- **Nonce mismatch** — usually a bug in how you're storing/reading the nonce in the session.
- **Token validation fails** — check your `ENTRA_TENANT_ID` is `common` if your app is multi-tenant, or your specific tenant ID if single-tenant.
- **"Cannot find module 'iron-session'" or similar** — make sure you're running commands from `apps/web/`, not the repo root.

---

## What you have at end of Day 2

A working multi-tenant-ready Next.js app with real Entra SSO, deployed to localhost. No directory features yet, but **the auth foundation that everything else hangs off is solid and verified**.

---

## What's next

The remaining Sprint 1 tasks from the roadmap:

- Push to GitHub (you've already created the repo)
- Connect Vercel to the repo, set env vars in Vercel, deploy `staging.skillsdirectory.com`
- Add Sentry DSN, verify error reporting
- GitHub Actions CI: lint + typecheck + test

Sprint 2 starts the real product:

- Tenant resolution middleware (look up your Entra `tid` in the control plane, route to the right tenant DB)
- The Prisma extension that auto-injects `tenant_id` filters
- First Graph API call (`/me`) on sign-in to populate the `users` table
- Profile photo download and blob storage

---

## A founder's honest warning

Days 1 and 2 will probably take longer than the time estimates above the first time you do this. Plan for a full weekend or four evenings. The friction points won't be the code — they'll be:

- **Fighting Entra app registration** — the portal UX is genuinely confusing, and the docs assume you know which permission ID maps to which scope.
- **TLS/cookie issues on localhost** — Iron-session in Lax-cookie mode is fine, but if you accidentally use a different domain you'll waste an hour.
- **Prisma migration paths with two schemas** — the dual-schema setup is unusual and you'll fight it once. After that it's fine.

Don't be discouraged when one of these eats half a day. That's not a sign you're doing it wrong — that's the cost of building B2B SaaS infrastructure properly.

When you hit a wall, **paste the exact error and the file you're working in** and someone (or your future self looking at this doc) can debug. Don't go round in circles for more than 30 minutes — ask.

---

## Notes on Node version

You're running Node v25.2.1. That's fine for development. The `"engines": { "node": ">=22 <23" }` field in `apps/web/package.json` tells Vercel to use Node 22 LTS in production. This means:

- **Local dev:** Node 25 (your current version)
- **Vercel production:** Node 22 LTS
- **CI (GitHub Actions):** match Vercel — set `node-version: '22'` in workflow files

If you ever see "works locally, breaks in Vercel" — Node version drift is the first thing to check.
