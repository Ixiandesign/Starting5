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
- **Lineup** — a set of exactly 6 item picks (one per slot) from a single category, created by
  an account holder. Has one `explanation` and a `status` of `draft` or `published`.
- **Slot** — one of the 6 fixed positions every lineup has, regardless of category: `PG`, `SG`,
  `SF`, `PF`, `C`, `SIXTH_MAN`. Every category is lineup-shaped — that's the "Starting 5+1"
  conceit the app is named for.
- **Explanation** — required free-text field (10-1000 chars) on a lineup justifying the picks
  as a whole; shown wherever the lineup is displayed or voted on.
- **Draft** — a lineup not yet published: private to its owner, freely editable/deletable, not
  votable or enterable into a tournament/matchup.
- **Published** — a lineup made public (one-way transition from draft): visible in feed/profile
  and votable; editable/deletable by its owner only until it's entered into a tournament or
  matchup, at which point it's locked for integrity.
- **Match** — a single 1v1 vote between two lineups, with an open/closed `status`, a voting
  window (`opens_at`/`closes_at`), and a `winner_id` once closed. The atomic unit both
  standalone voting and tournament rounds are built from.
- **Quick 1v1** — not a separate feature: a tournament with `bracket_size = 2`, i.e. exactly one
  match. Reuses the tournament creation flow rather than its own code path.
- **Vote** — a single `(match_id, voter_id|fingerprint, choice_lineup_id)` row, insert-only via
  the `cast_vote()` RPC. Never updated or deleted — permanent by construction.
- **Fingerprint** — a random token in a signed, httpOnly cookie identifying an anonymous voter
  for dedup purposes. Not device/browser fingerprinting.
- **Tournament** — a single-elimination, randomly seeded bracket over a fixed `bracket_size`
  (`2`/`4`/`8`/`16`/`32`/`64`), with a `category_mode` (`single_category` | `cross_category`)
  and a `visibility` (`public` | `invite_only`).
- **Round** — one layer of a tournament's bracket; all its matches share the same voting window.
- **Bye** — an unfilled bracket slot from an under-full tournament. A first-round match with a
  bye never opens for voting — its human entrant auto-advances at seeding time.
- **Seeding** — the random assignment of entrants (and byes) into bracket slots, triggered at
  the signup deadline or an early creator-triggered start.
- **Comment** — a threaded (max depth 3), account-gated reply attached to a tournament (never to
  an individual match/round — a quick 1v1 is itself a tournament, so it already has one thread).
- **Report** — a flag an account holder raises on a comment for admin review. No automated
  hide threshold for MVP.

## Architecture decisions

- [ADR-0001](docs/adr/0001-auth-session-and-profile-creation.md) — Auth session handling &
  profile creation
- [ADR-0002](docs/adr/0002-lineup-structure-and-lifecycle.md) — Lineup structure & lifecycle
- [ADR-0003](docs/adr/0003-voting-and-matchup-mechanics.md) — Voting & matchup mechanics
- [ADR-0004](docs/adr/0004-tournament-lifecycle.md) — Tournament lifecycle
- [ADR-0005](docs/adr/0005-comments-and-moderation.md) — Comments & moderation
