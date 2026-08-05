# Phase 1: Core Data Model & Auth — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the full Supabase schema (profiles, categories/items, lineups, tournaments/matches, votes, comments, category suggestions) with RLS, wire Supabase Auth (email/password + Google OAuth) into the Next.js app, and seed the 4 launch categories — so Phase 2 (Lineup Builder) has a real, authenticated, populated backend to build against.

**Architecture:** Supabase Postgres migrations (SQL files under `supabase/migrations/`) define every table from the approved spec's data model, each with RLS policies applied in the same file. Vote writes go through a single `SECURITY DEFINER` Postgres function (`cast_vote`) instead of direct table inserts, so anonymous-vote dedup can't be bypassed by a crafted client request. `@supabase/ssr` provides browser/server Supabase clients plus middleware-based session refresh. A minimal auth UI (sign up, sign in, sign out, OAuth callback) proves the auth wiring end-to-end via a real Playwright test run against the actual dev Supabase project — there is no local Docker/Postgres available on this machine, so all development and testing happens directly against the linked remote dev project.

**Tech Stack:** Next.js App Router, TypeScript, `@supabase/ssr` + `@supabase/supabase-js`, Supabase CLI (already linked), Vitest, Playwright.

## Global Constraints

- No Docker available locally — `supabase start` (local Postgres) is not used. Every migration is applied with `supabase db push` directly to the linked remote dev project (ref `afufdejwicariblnplfm`, region `us-east-1`, org `fymrweegzbxcbljkbvzb`).
- Env vars already exist in `.env.local` (gitignored): `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_DB_PASSWORD`. `.env.example` (committed) lists the three app-facing names with empty values.
- Every table gets `alter table ... enable row level security;` — no table ships without RLS in the same migration that creates it.
- Lineup slot positions are exactly `PG | SG | SF | PF | C | SIXTH_MAN`, applied as thematic labels to every category (spec decision #2).
- A lineup's 6 slots all come from one `category_id` (spec decision #1) — enforced at the app layer in Phase 2 (item pickers only offer items from the lineup's category); this phase just stores the FK.
- Vote writes never go through a direct `insert into votes` from a client role — only through `cast_vote()` (spec decision #5, #4).
- TDD: for SQL, "test" means a Vitest integration test (`src/**/*.integration.test.ts`, excluded from the default `npm test` / CI `unit-tests` job, run manually via `npm run test:integration`) that hits the real linked Supabase project with the anon key and asserts RLS behavior. For the auth flow, "test" means the Playwright e2e test in Task 11, which is added to the CI `e2e-tests` job and therefore needs Supabase secrets wired into CI (Task 1).

---

### Task 1: Wire Supabase env vars into CI

**Files:**

- Modify: `.github/workflows/ci.yml`

**Interfaces:**

- Produces: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` available as env vars inside the `e2e-tests` and `build` jobs (read from GitHub repo secrets of the same name).

- [ ] **Step 1: Add repo secrets from the values already in `.env.local`**

Run (values come from `.env.local` in this repo — do not print them):

```bash
gh secret set NEXT_PUBLIC_SUPABASE_URL --body "$(grep '^NEXT_PUBLIC_SUPABASE_URL=' .env.local | cut -d= -f2-)"
gh secret set NEXT_PUBLIC_SUPABASE_ANON_KEY --body "$(grep '^NEXT_PUBLIC_SUPABASE_ANON_KEY=' .env.local | cut -d= -f2-)"
gh secret set SUPABASE_SERVICE_ROLE_KEY --body "$(grep '^SUPABASE_SERVICE_ROLE_KEY=' .env.local | cut -d= -f2-)"
```

Expected: three `gh secret set` confirmations, no errors.

- [ ] **Step 2: Reference the secrets in the `e2e-tests` and `build` jobs**

In `.github/workflows/ci.yml`, add an `env:` block to both the `e2e-tests` job and the `build` job (top level of the job, sibling to `steps:`):

```yaml
env:
  NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
  NEXT_PUBLIC_SUPABASE_ANON_KEY: ${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }}
```

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/ci.yml
git commit -m "chore(ci): wire Supabase env vars into e2e and build jobs"
```

---

### Task 2: Supabase client helpers + session-refresh middleware

**Files:**

- Create: `src/lib/supabase/client.ts`
- Create: `src/lib/supabase/server.ts`
- Create: `src/middleware.ts`
- Test: `src/lib/supabase/client.test.ts`

**Interfaces:**

- Produces: `createClient()` (browser, from `src/lib/supabase/client.ts`) and `createClient()` (server/async, from `src/lib/supabase/server.ts`) — both return a Supabase JS client typed against `Database` (added in Task 9; use `SupabaseClient` untyped for now, Task 9 will add the generic).

- [ ] **Step 1: Install packages**

```bash
npm install @supabase/ssr @supabase/supabase-js
```

- [ ] **Step 2: Write the failing test for the browser client factory**

```typescript
// src/lib/supabase/client.test.ts
import { describe, expect, it } from "vitest";
import { createClient } from "./client";

describe("createClient (browser)", () => {
  it("returns a Supabase client configured with the public env vars", () => {
    const supabase = createClient();
    expect(supabase.supabaseUrl).toBe(process.env.NEXT_PUBLIC_SUPABASE_URL);
    expect(supabase).toHaveProperty("auth");
    expect(supabase).toHaveProperty("from");
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `npm run test -- src/lib/supabase/client.test.ts`
Expected: FAIL — `Cannot find module './client'`

- [ ] **Step 4: Implement the browser client**

```typescript
// src/lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npm run test -- src/lib/supabase/client.test.ts`
Expected: PASS

- [ ] **Step 6: Implement the server client** (no unit test — depends on `next/headers`, which requires a request context; it's exercised by the Playwright test in Task 11)

```typescript
// src/lib/supabase/server.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options),
            );
          } catch {
            // Called from a Server Component — middleware (Step 7) refreshes the session instead.
          }
        },
      },
    },
  );
}
```

- [ ] **Step 7: Implement session-refresh middleware**

```typescript
// src/middleware.ts
import { createServerClient } from "@supabase/ssr";
import { type NextRequest, NextResponse } from "next/server";

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options),
          );
        },
      },
    },
  );

  await supabase.auth.getUser();

  return response;
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

- [ ] **Step 8: Verify build still passes**

Run: `npm run build`
Expected: succeeds (middleware compiles, no type errors)

- [ ] **Step 9: Commit**

```bash
git add src/lib/supabase/client.ts src/lib/supabase/client.test.ts src/lib/supabase/server.ts src/middleware.ts package.json package-lock.json
git commit -m "feat: add Supabase browser/server clients and session-refresh middleware"
```

---

### Task 3: Migration — `profiles`

**Files:**

- Create: `supabase/migrations/20260805100000_profiles.sql`

**Interfaces:**

- Produces: `public.profiles(id uuid pk, username text unique, avatar_url text, created_at timestamptz)`, auto-populated on every `auth.users` insert.

- [ ] **Step 1: Write the migration**

```sql
-- supabase/migrations/20260805100000_profiles.sql
create table public.profiles (
  id uuid primary key references auth.users (id) on delete cascade,
  username text not null unique,
  avatar_url text,
  created_at timestamptz not null default now()
);

alter table public.profiles enable row level security;

create policy "profiles are publicly readable"
  on public.profiles for select
  using (true);

create policy "users can update their own profile"
  on public.profiles for update
  using (auth.uid() = id);

create or replace function public.handle_new_user()
returns trigger
language plpgsql
security definer
set search_path = public
as $$
begin
  insert into public.profiles (id, username)
  values (
    new.id,
    coalesce(new.raw_user_meta_data ->> 'username', 'user_' || substr(new.id::text, 1, 8))
  );
  return new;
end;
$$;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute function public.handle_new_user();
```

- [ ] **Step 2: Push the migration**

Run: `supabase db push`
Expected: `Applying migration 20260805100000_profiles.sql...` then success.

- [ ] **Step 3: Write the integration test**

```typescript
// src/lib/supabase/profiles.integration.test.ts
import { describe, expect, it } from "vitest";
import { createClient as createServiceClient } from "@supabase/supabase-js";

const admin = createServiceClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
);

describe("profiles auto-creation", () => {
  it("creates a profile row when a new auth user is created", async () => {
    const email = `test-${Date.now()}@example.com`;
    const { data: created, error: createError } = await admin.auth.admin.createUser({
      email,
      password: "test-password-123",
      email_confirm: true,
    });
    expect(createError).toBeNull();

    const { data: profile, error: profileError } = await admin
      .from("profiles")
      .select("id, username")
      .eq("id", created.user!.id)
      .single();

    expect(profileError).toBeNull();
    expect(profile?.id).toBe(created.user!.id);
    expect(profile?.username).toMatch(/^user_/);

    await admin.auth.admin.deleteUser(created.user!.id);
  });
});
```

- [ ] **Step 4: Run the integration test**

Run: `npm run test:integration -- src/lib/supabase/profiles.integration.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260805100000_profiles.sql src/lib/supabase/profiles.integration.test.ts
git commit -m "feat(db): add profiles table with auto-creation trigger"
```

---

### Task 4: Migration — `categories`, `items`

**Files:**

- Create: `supabase/migrations/20260805100100_categories_items.sql`

**Interfaces:**

- Produces: `public.categories(id, slug, name, description, status, created_by, created_at)`, `public.items(id, category_id, name, image_url, blurb, created_at)`.

- [ ] **Step 1: Write the migration**

```sql
-- supabase/migrations/20260805100100_categories_items.sql
create type public.category_status as enum ('active', 'suggested', 'rejected');

create table public.categories (
  id uuid primary key default gen_random_uuid(),
  slug text not null unique,
  name text not null,
  description text,
  status public.category_status not null default 'active',
  created_by uuid references public.profiles (id),
  created_at timestamptz not null default now()
);

alter table public.categories enable row level security;

create policy "active categories are publicly readable"
  on public.categories for select
  using (status = 'active');

create table public.items (
  id uuid primary key default gen_random_uuid(),
  category_id uuid not null references public.categories (id) on delete cascade,
  name text not null,
  image_url text,
  blurb text,
  created_at timestamptz not null default now()
);

alter table public.items enable row level security;

create policy "items in active categories are publicly readable"
  on public.items for select
  using (exists (
    select 1 from public.categories c
    where c.id = items.category_id and c.status = 'active'
  ));
```

- [ ] **Step 2: Push the migration**

Run: `supabase db push`
Expected: success.

- [ ] **Step 3: Write the integration test**

```typescript
// src/lib/supabase/categories.integration.test.ts
import { describe, expect, it } from "vitest";
import { createClient as createServiceClient } from "@supabase/supabase-js";
import { createClient as createAnonClient } from "@supabase/supabase-js";

const admin = createServiceClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
);
const anon = createAnonClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
);

describe("categories/items RLS", () => {
  it("hides suggested categories from anonymous reads but shows active ones", async () => {
    const { data: active } = await admin
      .from("categories")
      .insert({ slug: `active-${Date.now()}`, name: "Active Test", status: "active" })
      .select()
      .single();
    const { data: suggested } = await admin
      .from("categories")
      .insert({ slug: `suggested-${Date.now()}`, name: "Suggested Test", status: "suggested" })
      .select()
      .single();

    const { data: readAsAnon } = await anon
      .from("categories")
      .select("id")
      .in("id", [active!.id, suggested!.id]);

    expect(readAsAnon?.map((c) => c.id)).toEqual([active!.id]);

    await admin.from("categories").delete().in("id", [active!.id, suggested!.id]);
  });
});
```

- [ ] **Step 4: Run the integration test**

Run: `npm run test:integration -- src/lib/supabase/categories.integration.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260805100100_categories_items.sql src/lib/supabase/categories.integration.test.ts
git commit -m "feat(db): add categories and items tables with status-gated RLS"
```

---

### Task 5: Migration — `lineups`, `lineup_slots`

**Files:**

- Create: `supabase/migrations/20260805100200_lineups.sql`

**Interfaces:**

- Produces: `public.lineups(id, owner_id, category_id, name, explanation, created_at, updated_at)`, `public.lineup_slots(id, lineup_id, position, item_id)` where `position` is one of `PG|SG|SF|PF|C|SIXTH_MAN`.

- [ ] **Step 1: Write the migration**

```sql
-- supabase/migrations/20260805100200_lineups.sql
create type public.lineup_position as enum ('PG', 'SG', 'SF', 'PF', 'C', 'SIXTH_MAN');

create table public.lineups (
  id uuid primary key default gen_random_uuid(),
  owner_id uuid not null references public.profiles (id) on delete cascade,
  category_id uuid not null references public.categories (id),
  name text not null,
  explanation text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

alter table public.lineups enable row level security;

create policy "lineups are publicly readable"
  on public.lineups for select using (true);

create policy "owners can insert their own lineups"
  on public.lineups for insert with check (auth.uid() = owner_id);

create policy "owners can update their own lineups"
  on public.lineups for update using (auth.uid() = owner_id);

create policy "owners can delete their own lineups"
  on public.lineups for delete using (auth.uid() = owner_id);

create table public.lineup_slots (
  id uuid primary key default gen_random_uuid(),
  lineup_id uuid not null references public.lineups (id) on delete cascade,
  position public.lineup_position not null,
  item_id uuid not null references public.items (id),
  unique (lineup_id, position)
);

alter table public.lineup_slots enable row level security;

create policy "lineup slots are publicly readable"
  on public.lineup_slots for select using (true);

create policy "owners manage their own lineup slots"
  on public.lineup_slots for all
  using (exists (
    select 1 from public.lineups l
    where l.id = lineup_slots.lineup_id and l.owner_id = auth.uid()
  ))
  with check (exists (
    select 1 from public.lineups l
    where l.id = lineup_slots.lineup_id and l.owner_id = auth.uid()
  ));
```

- [ ] **Step 2: Push the migration**

Run: `supabase db push`
Expected: success.

- [ ] **Step 3: Write the integration test**

```typescript
// src/lib/supabase/lineups.integration.test.ts
import { describe, expect, it } from "vitest";
import { createClient as createServiceClient } from "@supabase/supabase-js";

const admin = createServiceClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
);

describe("lineups/lineup_slots", () => {
  it("rejects a second item in the same slot position for one lineup", async () => {
    const { data: user } = await admin.auth.admin.createUser({
      email: `lineup-test-${Date.now()}@example.com`,
      password: "test-password-123",
      email_confirm: true,
    });
    const { data: category } = await admin
      .from("categories")
      .insert({ slug: `cat-${Date.now()}`, name: "Test Cat" })
      .select()
      .single();
    const { data: item1 } = await admin
      .from("items")
      .insert({ category_id: category!.id, name: "Item 1" })
      .select()
      .single();
    const { data: item2 } = await admin
      .from("items")
      .insert({ category_id: category!.id, name: "Item 2" })
      .select()
      .single();
    const { data: lineup } = await admin
      .from("lineups")
      .insert({ owner_id: user!.user!.id, category_id: category!.id, name: "Test Lineup" })
      .select()
      .single();

    const { error: firstInsert } = await admin
      .from("lineup_slots")
      .insert({ lineup_id: lineup!.id, position: "PG", item_id: item1!.id });
    expect(firstInsert).toBeNull();

    const { error: duplicateSlot } = await admin
      .from("lineup_slots")
      .insert({ lineup_id: lineup!.id, position: "PG", item_id: item2!.id });
    expect(duplicateSlot?.code).toBe("23505");

    await admin.from("lineups").delete().eq("id", lineup!.id);
    await admin.from("items").delete().in("id", [item1!.id, item2!.id]);
    await admin.from("categories").delete().eq("id", category!.id);
    await admin.auth.admin.deleteUser(user!.user!.id);
  });
});
```

- [ ] **Step 4: Run the integration test**

Run: `npm run test:integration -- src/lib/supabase/lineups.integration.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260805100200_lineups.sql src/lib/supabase/lineups.integration.test.ts
git commit -m "feat(db): add lineups and lineup_slots tables"
```

---

### Task 6: Migration — `tournaments`, `tournament_entries`, `matches`

**Files:**

- Create: `supabase/migrations/20260805100300_tournaments.sql`

**Interfaces:**

- Produces: `public.tournaments(id, creator_id, name, mode, category_id, bracket_size, signup_deadline, round_duration, status, invite_code, created_at)`, `public.tournament_entries(id, tournament_id, lineup_id, seed, joined_at)`, `public.matches(id, tournament_id, round, slot_in_round, entry_a_id, entry_b_id, winner_entry_id, voting_opens_at, voting_closes_at, status, created_at)`.

- [ ] **Step 1: Write the migration**

```sql
-- supabase/migrations/20260805100300_tournaments.sql
create type public.tournament_mode as enum ('single_category', 'open');
create type public.tournament_status as enum ('draft', 'open', 'in_progress', 'complete');
create type public.match_status as enum ('pending', 'open', 'closed');

create table public.tournaments (
  id uuid primary key default gen_random_uuid(),
  creator_id uuid not null references public.profiles (id),
  name text not null,
  mode public.tournament_mode not null,
  category_id uuid references public.categories (id),
  bracket_size int not null,
  signup_deadline timestamptz not null,
  round_duration interval not null,
  status public.tournament_status not null default 'draft',
  invite_code text not null unique default encode(gen_random_bytes(6), 'hex'),
  created_at timestamptz not null default now(),
  constraint bracket_size_power_of_two check (bracket_size >= 2 and (bracket_size & (bracket_size - 1)) = 0),
  constraint single_category_requires_category check (mode <> 'single_category' or category_id is not null)
);

alter table public.tournaments enable row level security;

create policy "tournaments are publicly readable"
  on public.tournaments for select using (true);

create policy "creators can insert tournaments"
  on public.tournaments for insert with check (auth.uid() = creator_id);

create policy "creators can update their tournaments"
  on public.tournaments for update using (auth.uid() = creator_id);

create table public.tournament_entries (
  id uuid primary key default gen_random_uuid(),
  tournament_id uuid not null references public.tournaments (id) on delete cascade,
  lineup_id uuid not null references public.lineups (id),
  seed int,
  joined_at timestamptz not null default now(),
  unique (tournament_id, lineup_id)
);

alter table public.tournament_entries enable row level security;

create policy "entries are publicly readable"
  on public.tournament_entries for select using (true);

create policy "lineup owners can enter their own lineup"
  on public.tournament_entries for insert
  with check (exists (
    select 1 from public.lineups l
    where l.id = tournament_entries.lineup_id and l.owner_id = auth.uid()
  ));

create table public.matches (
  id uuid primary key default gen_random_uuid(),
  tournament_id uuid not null references public.tournaments (id) on delete cascade,
  round int not null,
  slot_in_round int not null,
  entry_a_id uuid references public.tournament_entries (id),
  entry_b_id uuid references public.tournament_entries (id),
  winner_entry_id uuid references public.tournament_entries (id),
  voting_opens_at timestamptz,
  voting_closes_at timestamptz,
  status public.match_status not null default 'pending',
  created_at timestamptz not null default now(),
  unique (tournament_id, round, slot_in_round)
);

alter table public.matches enable row level security;

create policy "matches are publicly readable"
  on public.matches for select using (true);
```

Note: `matches` has no client-facing insert/update policy — bracket generation and round advancement (Phase 4) run through server-side code using the service role key, which bypasses RLS by design.

- [ ] **Step 2: Push the migration**

Run: `supabase db push`
Expected: success.

- [ ] **Step 3: Write the integration test**

```typescript
// src/lib/supabase/tournaments.integration.test.ts
import { describe, expect, it } from "vitest";
import { createClient as createServiceClient } from "@supabase/supabase-js";

const admin = createServiceClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
);

describe("tournaments constraints", () => {
  it("rejects a bracket_size that is not a power of two", async () => {
    const { data: user } = await admin.auth.admin.createUser({
      email: `tourney-test-${Date.now()}@example.com`,
      password: "test-password-123",
      email_confirm: true,
    });

    const { error } = await admin.from("tournaments").insert({
      creator_id: user!.user!.id,
      name: "Bad Bracket",
      mode: "open",
      bracket_size: 6,
      signup_deadline: new Date(Date.now() + 86_400_000).toISOString(),
      round_duration: "24 hours",
    });

    expect(error?.code).toBe("23514");

    await admin.auth.admin.deleteUser(user!.user!.id);
  });

  it("rejects single_category mode without a category_id", async () => {
    const { data: user } = await admin.auth.admin.createUser({
      email: `tourney-test-${Date.now()}-2@example.com`,
      password: "test-password-123",
      email_confirm: true,
    });

    const { error } = await admin.from("tournaments").insert({
      creator_id: user!.user!.id,
      name: "Missing Category",
      mode: "single_category",
      bracket_size: 8,
      signup_deadline: new Date(Date.now() + 86_400_000).toISOString(),
      round_duration: "24 hours",
    });

    expect(error?.code).toBe("23514");

    await admin.auth.admin.deleteUser(user!.user!.id);
  });
});
```

- [ ] **Step 4: Run the integration test**

Run: `npm run test:integration -- src/lib/supabase/tournaments.integration.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260805100300_tournaments.sql src/lib/supabase/tournaments.integration.test.ts
git commit -m "feat(db): add tournaments, tournament_entries, and matches tables"
```

---

### Task 7: Migration — `votes` + `cast_vote()`

**Files:**

- Create: `supabase/migrations/20260805100400_votes.sql`

**Interfaces:**

- Produces: `public.votes(id, match_id, voter_id, voter_fingerprint, choice_entry_id, created_at)`; `public.cast_vote(p_match_id uuid, p_choice_entry_id uuid, p_fingerprint text default null) returns public.votes` — the only sanctioned write path for votes.
- Consumes: `public.matches` (Task 6), `public.tournament_entries` (Task 6).

- [ ] **Step 1: Write the migration**

```sql
-- supabase/migrations/20260805100400_votes.sql
create table public.votes (
  id uuid primary key default gen_random_uuid(),
  match_id uuid not null references public.matches (id) on delete cascade,
  voter_id uuid references public.profiles (id),
  voter_fingerprint text,
  choice_entry_id uuid not null references public.tournament_entries (id),
  created_at timestamptz not null default now(),
  constraint voter_or_fingerprint check (voter_id is not null or voter_fingerprint is not null)
);

create unique index votes_match_voter_unique
  on public.votes (match_id, voter_id)
  where voter_id is not null;

create unique index votes_match_fingerprint_unique
  on public.votes (match_id, voter_fingerprint)
  where voter_id is null and voter_fingerprint is not null;

alter table public.votes enable row level security;

create policy "vote tallies are publicly readable"
  on public.votes for select using (true);

-- No insert/update/delete policies: all writes go through cast_vote(), which
-- runs as SECURITY DEFINER and therefore bypasses RLS by design.

create or replace function public.cast_vote(
  p_match_id uuid,
  p_choice_entry_id uuid,
  p_fingerprint text default null
)
returns public.votes
language plpgsql
security definer
set search_path = public
as $$
declare
  v_voter_id uuid := auth.uid();
  v_match public.matches;
  v_vote public.votes;
begin
  select * into v_match from public.matches where id = p_match_id;

  if v_match is null then
    raise exception 'match not found';
  end if;

  if v_match.status <> 'open' then
    raise exception 'match is not open for voting';
  end if;

  if p_choice_entry_id is distinct from v_match.entry_a_id
     and p_choice_entry_id is distinct from v_match.entry_b_id then
    raise exception 'choice is not an entrant in this match';
  end if;

  if v_voter_id is null and p_fingerprint is null then
    raise exception 'anonymous votes require a fingerprint';
  end if;

  insert into public.votes (match_id, voter_id, voter_fingerprint, choice_entry_id)
  values (
    p_match_id,
    v_voter_id,
    case when v_voter_id is null then p_fingerprint else null end,
    p_choice_entry_id
  )
  returning * into v_vote;

  return v_vote;
end;
$$;

grant execute on function public.cast_vote(uuid, uuid, text) to anon, authenticated;
```

- [ ] **Step 2: Push the migration**

Run: `supabase db push`
Expected: success.

- [ ] **Step 3: Write the integration test**

```typescript
// src/lib/supabase/votes.integration.test.ts
import { describe, expect, it } from "vitest";
import { createClient as createServiceClient } from "@supabase/supabase-js";
import { createClient as createAnonClient } from "@supabase/supabase-js";

const admin = createServiceClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
);
const anon = createAnonClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
);

async function seedOpenMatch() {
  const { data: user } = await admin.auth.admin.createUser({
    email: `vote-test-${Date.now()}@example.com`,
    password: "test-password-123",
    email_confirm: true,
  });
  const { data: category } = await admin
    .from("categories")
    .insert({ slug: `vote-cat-${Date.now()}`, name: "Vote Test Cat" })
    .select()
    .single();
  const { data: lineupA } = await admin
    .from("lineups")
    .insert({ owner_id: user!.user!.id, category_id: category!.id, name: "Team A" })
    .select()
    .single();
  const { data: lineupB } = await admin
    .from("lineups")
    .insert({ owner_id: user!.user!.id, category_id: category!.id, name: "Team B" })
    .select()
    .single();
  const { data: tournament } = await admin
    .from("tournaments")
    .insert({
      creator_id: user!.user!.id,
      name: "Vote Test Tournament",
      mode: "open",
      bracket_size: 2,
      signup_deadline: new Date().toISOString(),
      round_duration: "24 hours",
      status: "in_progress",
    })
    .select()
    .single();
  const { data: entryA } = await admin
    .from("tournament_entries")
    .insert({ tournament_id: tournament!.id, lineup_id: lineupA!.id, seed: 1 })
    .select()
    .single();
  const { data: entryB } = await admin
    .from("tournament_entries")
    .insert({ tournament_id: tournament!.id, lineup_id: lineupB!.id, seed: 2 })
    .select()
    .single();
  const { data: match } = await admin
    .from("matches")
    .insert({
      tournament_id: tournament!.id,
      round: 1,
      slot_in_round: 1,
      entry_a_id: entryA!.id,
      entry_b_id: entryB!.id,
      status: "open",
      voting_opens_at: new Date().toISOString(),
      voting_closes_at: new Date(Date.now() + 86_400_000).toISOString(),
    })
    .select()
    .single();

  return { match: match!, entryA: entryA!, userId: user!.user!.id };
}

describe("cast_vote", () => {
  it("lets an anonymous voter cast one vote per match via fingerprint", async () => {
    const { match, entryA } = await seedOpenMatch();
    const fingerprint = `fp-${Date.now()}`;

    const { data: vote, error } = await anon.rpc("cast_vote", {
      p_match_id: match.id,
      p_choice_entry_id: entryA.id,
      p_fingerprint: fingerprint,
    });
    expect(error).toBeNull();
    expect(vote.voter_fingerprint).toBe(fingerprint);

    const { error: secondVoteError } = await anon.rpc("cast_vote", {
      p_match_id: match.id,
      p_choice_entry_id: entryA.id,
      p_fingerprint: fingerprint,
    });
    expect(secondVoteError).not.toBeNull();
  });

  it("rejects a vote for an entry not in the match", async () => {
    const { match } = await seedOpenMatch();

    const { error } = await anon.rpc("cast_vote", {
      p_match_id: match.id,
      p_choice_entry_id: "00000000-0000-0000-0000-000000000000",
      p_fingerprint: `fp-${Date.now()}`,
    });

    expect(error).not.toBeNull();
  });

  it("rejects an anonymous vote with no fingerprint", async () => {
    const { match, entryA } = await seedOpenMatch();

    const { error } = await anon.rpc("cast_vote", {
      p_match_id: match.id,
      p_choice_entry_id: entryA.id,
    });

    expect(error).not.toBeNull();
  });
});
```

- [ ] **Step 4: Run the integration test**

Run: `npm run test:integration -- src/lib/supabase/votes.integration.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260805100400_votes.sql src/lib/supabase/votes.integration.test.ts
git commit -m "feat(db): add votes table and cast_vote security-definer function"
```

---

### Task 8: Migration — `comments`, `category_suggestions`

**Files:**

- Create: `supabase/migrations/20260805100500_comments_suggestions.sql`

**Interfaces:**

- Produces: `public.comments(id, tournament_id, author_id, body, created_at)`, `public.category_suggestions(id, suggested_by, name, description, status, created_at)`.

- [ ] **Step 1: Write the migration**

```sql
-- supabase/migrations/20260805100500_comments_suggestions.sql
create table public.comments (
  id uuid primary key default gen_random_uuid(),
  tournament_id uuid not null references public.tournaments (id) on delete cascade,
  author_id uuid not null references public.profiles (id),
  body text not null check (char_length(body) between 1 and 2000),
  created_at timestamptz not null default now()
);

alter table public.comments enable row level security;

create policy "comments are publicly readable"
  on public.comments for select using (true);

create policy "authenticated users can post comments"
  on public.comments for insert with check (auth.uid() = author_id);

create policy "authors can delete their own comments"
  on public.comments for delete using (auth.uid() = author_id);

create type public.suggestion_status as enum ('pending', 'approved', 'rejected');

create table public.category_suggestions (
  id uuid primary key default gen_random_uuid(),
  suggested_by uuid references public.profiles (id),
  name text not null,
  description text,
  status public.suggestion_status not null default 'pending',
  created_at timestamptz not null default now()
);

alter table public.category_suggestions enable row level security;

create policy "suggesters can view their own suggestions"
  on public.category_suggestions for select using (auth.uid() = suggested_by);

create policy "authenticated users can suggest a category"
  on public.category_suggestions for insert with check (auth.uid() = suggested_by);
```

- [ ] **Step 2: Push the migration**

Run: `supabase db push`
Expected: success.

- [ ] **Step 3: Write the integration test**

```typescript
// src/lib/supabase/comments.integration.test.ts
import { describe, expect, it } from "vitest";
import { createClient as createServiceClient } from "@supabase/supabase-js";

const admin = createServiceClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
);

describe("comments", () => {
  it("rejects a comment body longer than 2000 characters", async () => {
    const { data: user } = await admin.auth.admin.createUser({
      email: `comment-test-${Date.now()}@example.com`,
      password: "test-password-123",
      email_confirm: true,
    });
    const { data: category } = await admin
      .from("categories")
      .insert({ slug: `comment-cat-${Date.now()}`, name: "Comment Test Cat" })
      .select()
      .single();
    const { data: lineup } = await admin
      .from("lineups")
      .insert({ owner_id: user!.user!.id, category_id: category!.id, name: "Team" })
      .select()
      .single();
    const { data: tournament } = await admin
      .from("tournaments")
      .insert({
        creator_id: user!.user!.id,
        name: "Comment Test Tournament",
        mode: "open",
        bracket_size: 2,
        signup_deadline: new Date().toISOString(),
        round_duration: "24 hours",
      })
      .select()
      .single();

    const { error } = await admin.from("comments").insert({
      tournament_id: tournament!.id,
      author_id: user!.user!.id,
      body: "x".repeat(2001),
    });

    expect(error?.code).toBe("23514");

    await admin.from("tournaments").delete().eq("id", tournament!.id);
    await admin.from("lineups").delete().eq("id", lineup!.id);
    await admin.from("categories").delete().eq("id", category!.id);
    await admin.auth.admin.deleteUser(user!.user!.id);
  });
});
```

- [ ] **Step 4: Run the integration test**

Run: `npm run test:integration -- src/lib/supabase/comments.integration.test.ts`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add supabase/migrations/20260805100500_comments_suggestions.sql src/lib/supabase/comments.integration.test.ts
git commit -m "feat(db): add comments and category_suggestions tables"
```

---

### Task 9: Generate TypeScript DB types + add `test:integration` script

**Files:**

- Create: `src/lib/supabase/database.types.ts` (generated, not hand-edited)
- Modify: `src/lib/supabase/client.ts`
- Modify: `src/lib/supabase/server.ts`
- Modify: `package.json`
- Modify: `vitest.config.ts`

**Interfaces:**

- Produces: `Database` type, re-exported implicitly via the typed clients; both `createClient()` factories now return `SupabaseClient<Database>`.

- [ ] **Step 1: Generate types from the linked project**

```bash
supabase gen types typescript --project-id afufdejwicariblnplfm --schema public > src/lib/supabase/database.types.ts
```

- [ ] **Step 2: Type the clients against `Database`**

```typescript
// src/lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "./database.types";

export function createClient() {
  return createBrowserClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  );
}
```

Apply the same `<Database>` generic + `import type { Database } from "./database.types"` to `createServerClient` in `src/lib/supabase/server.ts`.

- [ ] **Step 3: Add `test:integration` npm script and a separate Vitest project so it's excluded from the default `npm test`**

```json
// package.json — add to "scripts"
"test:integration": "vitest run --config vitest.integration.config.ts"
```

```typescript
// vitest.integration.config.ts (new file, sibling to vitest.config.ts)
import { defineConfig } from "vitest/config";
import path from "node:path";

export default defineConfig({
  test: {
    environment: "node",
    include: ["src/**/*.integration.test.ts"],
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
});
```

Update `vitest.config.ts`'s `test.include` to explicitly stay `["src/**/*.test.{ts,tsx}"]` and add `test.exclude` entry `"src/**/*.integration.test.ts"` so the default `npm test` never picks up integration tests.

- [ ] **Step 4: Run everything to confirm the split works**

Run: `npm run test` — Expected: only the non-integration tests run (from Tasks 2–8, none — those are all `.integration.test.ts`), still passes with 0 or the Task-2 client test.
Run: `npm run test:integration` — Expected: all integration tests from Tasks 3–8 run and pass.
Run: `npm run typecheck` — Expected: passes with the new generated types in place.

- [ ] **Step 5: Commit**

```bash
git add src/lib/supabase/database.types.ts src/lib/supabase/client.ts src/lib/supabase/server.ts package.json vitest.config.ts vitest.integration.config.ts
git commit -m "feat: generate typed Supabase clients and split integration tests from unit tests"
```

---

### Task 10: Auth UI — sign up, sign in, sign out, OAuth callback

**Files:**

- Create: `src/app/auth/sign-up/page.tsx`
- Create: `src/app/auth/sign-in/page.tsx`
- Create: `src/app/auth/callback/route.ts`
- Create: `src/app/auth/sign-out/route.ts`
- Modify: `src/app/page.tsx` (add a minimal sign-in-state indicator so Playwright in Task 11 has something to assert on)

**Interfaces:**

- Consumes: `createClient()` from `src/lib/supabase/client.ts` (Task 2/9) in the two pages; `createClient()` from `src/lib/supabase/server.ts` in the two route handlers.

- [ ] **Step 1: Sign-up page**

```tsx
// src/app/auth/sign-up/page.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { createClient } from "@/lib/supabase/client";

export default function SignUpPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    const supabase = createClient();
    const { error } = await supabase.auth.signUp({ email, password });
    if (error) {
      setError(error.message);
      return;
    }
    router.push("/");
    router.refresh();
  }

  return (
    <main>
      <h1>Sign up</h1>
      <form onSubmit={handleSubmit}>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
          minLength={8}
        />
        <button type="submit">Create account</button>
      </form>
      {error && <p role="alert">{error}</p>}
    </main>
  );
}
```

- [ ] **Step 2: Sign-in page**

```tsx
// src/app/auth/sign-in/page.tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { createClient } from "@/lib/supabase/client";

export default function SignInPage() {
  const router = useRouter();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    const supabase = createClient();
    const { error } = await supabase.auth.signInWithPassword({ email, password });
    if (error) {
      setError(error.message);
      return;
    }
    router.push("/");
    router.refresh();
  }

  return (
    <main>
      <h1>Sign in</h1>
      <form onSubmit={handleSubmit}>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
        <button type="submit">Sign in</button>
      </form>
      {error && <p role="alert">{error}</p>}
    </main>
  );
}
```

- [ ] **Step 3: OAuth callback route (for Google OAuth's redirect)**

```typescript
// src/app/auth/callback/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");

  if (code) {
    const supabase = await createClient();
    await supabase.auth.exchangeCodeForSession(code);
  }

  return NextResponse.redirect(`${origin}/`);
}
```

- [ ] **Step 4: Sign-out route**

```typescript
// src/app/auth/sign-out/route.ts
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function POST(request: Request) {
  const supabase = await createClient();
  await supabase.auth.signOut();
  return NextResponse.redirect(new URL("/", request.url));
}
```

- [ ] **Step 5: Minimal sign-in-state indicator on the home page**

Read the current file first, then replace its body content with an auth-aware header:

```tsx
// src/app/page.tsx
import { createClient } from "@/lib/supabase/server";

export default async function Home() {
  const supabase = await createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();

  return (
    <main>
      <h1>Starting5</h1>
      {user ? (
        <div>
          <p data-testid="signed-in-as">Signed in as {user.email}</p>
          <form action="/auth/sign-out" method="post">
            <button type="submit">Sign out</button>
          </form>
        </div>
      ) : (
        <div>
          <a href="/auth/sign-in">Sign in</a>
          <a href="/auth/sign-up">Sign up</a>
        </div>
      )}
    </main>
  );
}
```

- [ ] **Step 6: Enable Google OAuth provider (manual dashboard step — not scriptable via CLI)**

In the Supabase dashboard (Authentication → Providers → Google) for project `afufdejwicariblnplfm`, enable Google and add the Client ID/Secret from a Google Cloud OAuth consent screen. This step needs the project owner's Google Cloud credentials, so it's called out here rather than automated — the email/password flow (Steps 1–5) works independently and is what Task 11 tests.

- [ ] **Step 7: Verify the build**

Run: `npm run build`
Expected: succeeds, `/auth/sign-up` and `/auth/sign-in` routes listed in the build output.

- [ ] **Step 8: Commit**

```bash
git add src/app/auth src/app/page.tsx
git commit -m "feat: add email/password auth UI and OAuth callback route"
```

---

### Task 11: Playwright e2e — sign up → signed-in state → sign out

**Files:**

- Create: `e2e/auth.spec.ts`

**Interfaces:**

- Consumes: the real dev Supabase project (via the running Next.js dev server's env vars) — this test creates a real (throwaway) user, which is acceptable for this project's dedicated dev database.

- [ ] **Step 1: Write the e2e test**

```typescript
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test";

test("a visitor can sign up, land signed in, and sign out", async ({ page }) => {
  const email = `e2e-${Date.now()}@example.com`;
  const password = "test-password-123";

  await page.goto("/auth/sign-up");
  await page.getByLabel("Email").fill(email);
  await page.getByLabel("Password").fill(password);
  await page.getByRole("button", { name: "Create account" }).click();

  await expect(page.getByTestId("signed-in-as")).toHaveText(`Signed in as ${email}`);

  await page.getByRole("button", { name: "Sign out" }).click();

  await expect(page.getByRole("link", { name: "Sign in" })).toBeVisible();
});
```

- [ ] **Step 2: Run it**

Run: `npm run test:e2e -- e2e/auth.spec.ts`
Expected: PASS. (This requires `email_confirm` to not block sign-in — Supabase's default dev project allows unconfirmed sign-in for a `signUp` immediately followed by a session; if the test fails on "Signed in as" not appearing, check the project's Authentication → Settings → "Confirm email" toggle and disable it for this dev project.)

- [ ] **Step 3: Commit**

```bash
git add e2e/auth.spec.ts
git commit -m "test(e2e): cover sign-up to sign-out flow"
```

---

### Task 12: Seed the 4 launch categories

**Files:**

- Create: `supabase/seed.sql`

**Interfaces:**

- Produces: 4 rows in `public.categories` (`nba-players`, `sitcom-characters`, `politicians`, `inanimate-objects`), each with roughly 24 `items` rows.

- [ ] **Step 1: Write the seed file**

Draft ~24 items per category. Politicians is scoped to a bipartisan spread of past U.S. presidents (not current sitting officials) to keep the category evergreen and not politically inflammatory — flag this choice for the project owner's review in the commit/PR description.

```sql
-- supabase/seed.sql
insert into public.categories (slug, name, description, status) values
  ('nba-players', 'NBA Players', 'Pro basketball, any era.', 'active'),
  ('sitcom-characters', 'Sitcom Characters', 'TV sitcom characters, any show.', 'active'),
  ('politicians', 'Politicians', 'U.S. presidents, any era.', 'active'),
  ('inanimate-objects', 'Inanimate Objects', 'Everyday objects, personified.', 'active');

insert into public.items (category_id, name)
select id, name from public.categories, unnest(array[
  'LeBron James', 'Michael Jordan', 'Stephen Curry', 'Magic Johnson', 'Larry Bird',
  'Kareem Abdul-Jabbar', 'Shaquille O''Neal', 'Kobe Bryant', 'Tim Duncan', 'Kevin Durant',
  'Giannis Antetokounmpo', 'Nikola Jokic', 'Larry Bird', 'Hakeem Olajuwon', 'Charles Barkley',
  'Allen Iverson', 'Dirk Nowitzki', 'Dwyane Wade', 'Chris Paul', 'Russell Westbrook',
  'James Harden', 'Kevin Garnett', 'Scottie Pippen', 'John Stockton'
]) as name
where slug = 'nba-players'
on conflict do nothing;

insert into public.items (category_id, name)
select id, name from public.categories, unnest(array[
  'Michael Scott', 'Jim Halpert', 'Leslie Knope', 'Ron Swanson', 'Liz Lemon',
  'Jack Donaghy', 'Sheldon Cooper', 'Barney Stinson', 'Ted Mosby', 'Phoebe Buffay',
  'Ross Geller', 'Homer Simpson', 'Peter Griffin', 'Frasier Crane', 'Kramer',
  'George Costanza', 'Dwight Schrute', 'Andy Dwyer', 'April Ludgate', 'Chandler Bing',
  'Jerry Seinfeld', 'Cosmo Kramer', 'Michael Bluth', 'Tobias Funke'
]) as name
where slug = 'sitcom-characters'
on conflict do nothing;

insert into public.items (category_id, name)
select id, name from public.categories, unnest(array[
  'George Washington', 'Thomas Jefferson', 'Abraham Lincoln', 'Theodore Roosevelt', 'Franklin D. Roosevelt',
  'John F. Kennedy', 'Ronald Reagan', 'Dwight D. Eisenhower', 'Harry S. Truman', 'Woodrow Wilson',
  'Barack Obama', 'Bill Clinton', 'George H. W. Bush', 'Jimmy Carter', 'Gerald Ford',
  'Richard Nixon', 'Lyndon B. Johnson', 'James Madison', 'Andrew Jackson', 'Ulysses S. Grant',
  'William McKinley', 'Grover Cleveland', 'James Monroe', 'John Adams'
]) as name
where slug = 'politicians'
on conflict do nothing;

insert into public.items (category_id, name)
select id, name from public.categories, unnest(array[
  'Toaster', 'Stapler', 'Traffic Cone', 'Rubber Duck', 'Fire Hydrant',
  'Vending Machine', 'Garden Gnome', 'Shopping Cart', 'Lava Lamp', 'Beanbag Chair',
  'Swiss Army Knife', 'Disco Ball', 'Snow Globe', 'Whoopee Cushion', 'Traffic Light',
  'Parking Meter', 'Umbrella', 'Boombox', 'Pinball Machine', 'Jukebox',
  'Grandfather Clock', 'Weather Vane', 'Mailbox', 'Wind Chime'
]) as name
where slug = 'inanimate-objects'
on conflict do nothing;
```

- [ ] **Step 2: Apply the seed to the linked project**

```bash
supabase db execute --file supabase/seed.sql --linked
```

Expected: no errors; 4 categories and ~96 items inserted.

- [ ] **Step 3: Verify counts**

```bash
supabase db execute --linked --command "select c.slug, count(i.id) from public.categories c left join public.items i on i.category_id = c.id group by c.slug;"
```

Expected: 4 rows, each with an item count in the low-to-mid 20s.

- [ ] **Step 4: Commit**

```bash
git add supabase/seed.sql
git commit -m "feat(db): seed the 4 launch categories with starter items"
```

Note in the PR/commit description: the politicians list is scoped to past U.S. presidents specifically to sidestep current-politics controversy — flag for the project owner to confirm this framing (or adjust it) before Phase 6 launch.

---

## Definition of Done for Phase 1

- [ ] All 6 migrations pushed to the linked Supabase project and committed under `supabase/migrations/`.
- [ ] `npm run test` (unit) and `npm run test:integration` (RLS/DB integration) both pass locally.
- [ ] `npm run test:e2e` passes locally, including the new `e2e/auth.spec.ts`.
- [ ] CI is green on the PR that lands this work (`lint-typecheck-format`, `unit-tests`, `e2e-tests`, `build`), with Supabase secrets wired in from Task 1.
- [ ] 4 categories with ~24 items each are live in the dev Supabase project.
- [ ] Project owner has reviewed the seeded item lists (especially `politicians`) and the Google OAuth provider is enabled in the Supabase dashboard, or explicitly deferred to a later phase.
