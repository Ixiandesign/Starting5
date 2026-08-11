# Starting5 — Context

Domain glossary for the Starting5 MVP. Decisions are resolved via the wayfinder map
([issue #1](https://github.com/Ixiandesign/Starting5/issues/1)) and recorded here as terms
settle. See `docs/adr/` for the architecturally significant decisions behind these terms.

## Glossary

- **Account** — an `auth.users` row (Supabase Auth) paired 1:1 with a `profiles` row. Created
  via email/password or Google OAuth.
- **Profile** — the `public.profiles` row for an account: `username`, `display_name`,
  `avatar_emoji`, `created_at`. Auto-created by a database trigger on first sign-in, never
  created client-side.
- **Anonymous visitor** — someone with no session. Can browse and vote. Cannot create a lineup,
  tournament, comment, or friend request — those require an account.
- **Username** — unique handle, 3-20 chars, lowercase `[a-z0-9_]`, used in profile URLs. Derived
  from the email local-part at signup (de-duplicated on collision), user-editable after.
- **Display name** — free-text label shown in the UI, 1-40 chars, defaults to username.
- **Session** — a Supabase Auth session held in `sb-*` cookies, refreshed on every request by
  `src/middleware.ts` via `@supabase/ssr`.

## Architecture decisions

- [ADR-0001](docs/adr/0001-auth-session-and-profile-creation.md) — Auth session handling &
  profile creation
