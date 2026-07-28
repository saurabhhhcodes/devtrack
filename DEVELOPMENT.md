# Development Guide

Everything you need to run DevTrack locally from scratch in under 10 minutes.

---

## Prerequisites

| Tool    | Version | Check           |
| ------- | ------- | --------------- |
| Node.js | >= 18   | `node -v`       |
| npm     | >= 9    | `npm -v`        |
| Git     | any     | `git --version` |

You also need free accounts on:

- [Supabase](https://supabase.com) — for the database
- GitHub — for OAuth (you already have this)

---

## 1. Clone and install

```bash
git clone https://github.com/Priyanshu-byte-coder/devtrack.git
cd devtrack
npm install
```

---

## 2. Set up Supabase

1. Go to [supabase.com](https://supabase.com) → **New Project**
2. Pick a name, region, and database password — save the password somewhere
3. Wait ~1 minute for project to provision
4. Go to **SQL Editor** → **New Query**
5. Paste the full contents of `supabase/schema.sql` and click **Run**
6. Go to **Project Settings → API** and copy three values:
   - **Project URL** → `NEXT_PUBLIC_SUPABASE_URL`
   - **anon / public** key → `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - **service_role** secret → `SUPABASE_SERVICE_ROLE_KEY`

> The `service_role` key has admin access. Never expose it client-side. DevTrack uses it only in server-side API routes.

---

## 3. Create a GitHub OAuth App

1. Go to [github.com/settings/applications/new](https://github.com/settings/applications/new)
2. Fill in:
   - **Application name:** `DevTrack (local)`
   - **Homepage URL:** `http://localhost:3000`
   - **Authorization callback URL:** `http://localhost:3000/api/auth/callback/github`
3. Click **Register application**
4. Copy **Client ID** → `GITHUB_ID`
5. Click **Generate a new client secret** → copy it → `GITHUB_SECRET`

---

## 4. Configure environment

```bash
cp .env.example .env.local
```

Open `.env.local` and fill in all values:

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxxxxxxxxxxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# NextAuth
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=generate_with_openssl_rand_base64_32

# GitHub OAuth
GITHUB_ID=Ov23...
GITHUB_SECRET=your_github_client_secret
```

Generate `NEXTAUTH_SECRET`:

```bash
# macOS / Linux
openssl rand -base64 32

# Windows PowerShell
[Convert]::ToBase64String((1..32 | ForEach-Object { Get-Random -Maximum 256 }))
```

---

## 5. Run the dev server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Click **Sign in with GitHub**.

---

## Project Structure

```
src/
├── app/
│   ├── api/
│   │   ├── auth/[...nextauth]/   # GitHub OAuth via NextAuth
│   │   ├── metrics/
│   │   │   ├── contributions/    # GET /api/metrics/contributions?days=30
│   │   │   ├── prs/              # GET /api/metrics/prs
│   │   │   ├── streak/           # GET /api/metrics/streak
│   │   │   └── repos/            # GET /api/metrics/repos?days=30
│   │   └── goals/                # GET + POST /api/goals
│   ├── dashboard/
│   │   └── page.tsx              # Dashboard layout — add new widgets here
│   └── page.tsx                  # Landing page
├── components/
│   ├── ContributionGraph.tsx     # Bar chart with time range selector
│   ├── PRMetrics.tsx             # PR stats card grid
│   ├── GoalTracker.tsx           # Weekly goals progress bars
│   ├── StreakTracker.tsx         # Current + longest commit streak
│   ├── TopRepos.tsx              # Most active repos ranked list
│   ├── DashboardHeader.tsx       # Top bar with user avatar + sign out
│   └── SignOutButton.tsx
├── lib/
│   ├── auth.ts                   # NextAuth config, GitHub scopes, Supabase upsert
│   └── supabase.ts               # Supabase admin client (server-only)
supabase/
└── schema.sql                    # DB schema — run once in Supabase SQL Editor
```

### How data flows

```
Browser → Next.js API route → GitHub API (with user's OAuth token)
                           → Supabase (for goals, user records)
```

All GitHub API calls use the signed-in user's OAuth token — stored in the session via NextAuth. No shared API key.

---

## Available scripts

| Command              | What it does                        |
| -------------------- | ----------------------------------- |
| `npm run dev`        | Start dev server at localhost:3000  |
| `npm run build`      | Production build                    |
| `npm run lint`       | ESLint                              |
| `npm run type-check` | TypeScript compiler check (no emit) |

Run lint and type-check before pushing:

```bash
npm run lint && npm run type-check
```

---

## Adding a new dashboard widget

1. Create `src/components/MyWidget.tsx` — use `"use client"`, fetch from your API route
2. Create `src/app/api/metrics/my-widget/route.ts` — add `export const dynamic = "force-dynamic"`, guard with `getServerSession`
3. Import and place in `src/app/dashboard/page.tsx`

Pattern for an API route:

```ts
import { getServerSession } from "next-auth";
import { authOptions } from "@/lib/auth";

export const dynamic = "force-dynamic";

export async function GET() {
  const session = await getServerSession(authOptions);
  if (!session?.accessToken) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }
  // fetch from GitHub API using session.accessToken
  // fetch from Supabase using session.githubId
}
```

---

## Common errors

### `NEXTAUTH_SECRET` missing

```
[next-auth][error][NO_SECRET]
```

Add `NEXTAUTH_SECRET` to `.env.local`.

---

### GitHub OAuth callback mismatch

```
The redirect_uri is not associated with this application
```

Ensure the **Authorization callback URL** in your GitHub OAuth App is exactly:
`http://localhost:3000/api/auth/callback/github`

---

### Supabase "relation does not exist"

```
relation "users" does not exist
```

You forgot to run `supabase/schema.sql`. Go to Supabase SQL Editor and run it.

---

### GitHub API rate limit

```
{ "message": "API rate limit exceeded" }
```

You hit the 30 requests/minute search API limit. Wait 1 minute. In production this won't happen for normal usage.

---

## Questions?

Open a [GitHub Discussion](https://github.com/Priyanshu-byte-coder/devtrack/discussions) — not an issue.
