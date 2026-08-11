# ADR-0001: Auth session handling & profile creation

## Status

Accepted — resolved via wayfinder ticket [#2](https://github.com/Ixiandesign/Starting5/issues/2).

## Context

Starting5 needs Supabase Auth (email/password + Google OAuth) wired into a Next.js App Router
app, a `profiles` row per account, and a clear line between what an anonymous visitor and an
account holder can each do. Stack and standing preferences (anonymous voting, account required
to create) are fixed by the wayfinder map ([#1](https://github.com/Ixiandesign/Starting5/issues/1)).

## Decision

- **Session handling**: `@supabase/ssr` — `createBrowserClient` in Client Components,
  `createServerClient` in Server Components / Route Handlers / Server Actions, both reading and
  writing the `sb-*` cookies. `src/middleware.ts` calls `supabase.auth.getUser()` on every
  request to refresh the session before it expires.
- **Profile creation**: a Postgres trigger (`on_auth_user_created` → `handle_new_user()`) inserts
  into `public.profiles` immediately after a row lands in `auth.users` — never a client-side
  call. `username` is derived from the email local-part, de-duplicated with a numeric suffix on
  collision; `display_name` and `avatar_emoji` start null and fall back to `username` / an
  initial in the UI until the user edits them.
- **Profile fields**: `username` (unique, 3-20 chars, `[a-z0-9_]`, lowercase-enforced),
  `display_name` (1-40 chars, optional), `avatar_emoji` (single emoji or short string, optional —
  text-only per the map's Out-of-scope on images), `created_at`.
- **RLS**: `profiles` — public read, owner-only update (`auth.uid() = id`).
- **Anonymous vs. account boundary**: anonymous visitors can browse and vote (dedup via
  fingerprint/cookie, per the map's standing preference). An account is required to create a
  lineup, create or enter a tournament, post a comment, or send/accept/decline a friend request.
- **Email verification**: password signups require confirmation before first sign-in (Supabase
  default); Google OAuth signups skip it since Google already verified the address.

## Consequences

- Profile rows can never be missing for a signed-in user — downstream tables can foreign-key to
  `profiles.id` without a nullable-profile edge case.
- Enabling Google OAuth requires a one-time manual step in the Supabase dashboard (Google Cloud
  OAuth credentials) — cannot be automated from this repo. Flagged for owner sign-off.
- Defaulting email verification to on adds signup friction; flagged for owner sign-off in case a
  lower-friction default is preferred for launch.

## References

- The pre-wayfinder Phase-1 plan (git history, commit `3b45274`) proposed the same trigger-based
  profile creation and `@supabase/ssr` approach — reaffirmed here independently, not copied
  wholesale.
