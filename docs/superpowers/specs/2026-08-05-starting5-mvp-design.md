# Starting5 — MVP Design & Phased Roadmap

**Date:** 2026-08-05
**Status:** Approved

## Context

Starting5 is a new open-source web app: users build a "Starting 5+1" (6-slot) lineup from a chosen category — NBA players, sitcom characters, politicians, inanimate objects, and more — write a short explanation for their picks, then pit lineups against each other in community-voted 1v1 matchups or bracket tournaments, including cross-category matchups (e.g. an NBA lineup vs. a Sitcom lineup). The project is funded via ads (the Monkeytype model: free, open source, ad-supported), licensed GPL-3.0 (matching Monkeytype), built on Supabase (Postgres/Auth/Storage) and hosted on Vercel.

This document covers the product design and phased roadmap from repo bootstrap through a shippable MVP. It was produced through a clarifying-question pass with the project owner; every numbered decision below reflects an explicit answer they gave, not an assumption, unless marked "(assumption)".

## Core Product Decisions

1. **Lineup = one category.** All 6 slots in a lineup come from the same category's item pool. Cross-category play happens at the matchup/tournament level, not inside one lineup.
2. **Fixed slot positions for every category:** PG, SG, SF, PF, C, 6th Man — used as thematic labels even for non-basketball categories (e.g. "who's the PG of The Office").
3. **Winner determination is pure community vote** — no stats/algorithm, same rule for same-category and cross-category matchups.
4. **One vote per voter per match, locked in** — no changing your vote once cast.
5. **Voting is anonymous-friendly:** browsing and voting require no account; creating a lineup, tournament, or comment requires an account (Supabase Auth). Anonymous votes are deduplicated via a browser fingerprint/cookie. (Assumption: this dedup is not bulletproof against a determined abuser — acceptable tradeoff for MVP given the explicit choice to allow anonymous voting.)
6. **Categories are curated for MVP** (a seeded list), with a lightweight "suggest a category" form feeding an admin-reviewed queue — no open self-serve category creation at MVP.
7. **Unified competition model:** a single `tournaments` table backs everything. A 1v1 "Quick Matchup" is just a `bracket_size = 2` tournament with one round — same voting/comment/invite machinery, with a thin dedicated UI on top so a casual 1v1 never shows bracket jargon. This was chosen over keeping matchups and tournaments as fully separate systems, to avoid duplicating voting/comment/invite logic twice.
8. **Tournament creation:** creator sets bracket size (power of 2: 2/4/8/16/32) and a mode chosen once at creation — **single-category** (all entrants must match one chosen category) or **open** (any category, matchups can cross categories any round).
9. **Entry/invites:** shareable invite link/code only for MVP — no in-app friend/notification system yet (explicitly planned for post-MVP). A creator shares the link either to get an existing lineup submitted or to prompt someone to build a new lineup for the tournament.
10. **Timer-based rounds with byes:** creator sets a signup deadline and a round voting duration. At the signup deadline (or a manual early start by the creator), unfilled bracket slots become byes — auto-advance, with higher seeds getting bye priority. Each round auto-closes at its deadline via a scheduled job; ties at deadline are broken by the higher seed advancing.
11. **No realtime for MVP** — vote counts/bracket state update on refresh/revalidation, not live sockets. Supabase Realtime is explicitly scoped as a post-MVP addition.
12. **Comments require an account**, scoped per tournament/matchup (a Quick Matchup is a tournament under the hood, so one comment system covers both).
13. **Ads:** Google AdSense. **License:** GPL-3.0, verified as Monkeytype's actual license (checked against `monkeytypegame/monkeytype`'s `LICENSE` file, which is GPLv3).
14. **Development process:** strict test-driven development throughout, and the build must include a way for a coding agent to exercise the app like a real user against actual screens/elements — both interactively during development and as a persisted automated e2e suite.

## Data Model (Supabase Postgres)

| Table                  | Purpose                                        | Key columns                                                                                                                                                                                                                                                                                                                               |
| ---------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `profiles`             | App-level user profile                         | `id` (FK `auth.users`), `username`, `avatar_url`, `created_at`                                                                                                                                                                                                                                                                            |
| `categories`           | A pool of items lineups draw from              | `id`, `slug`, `name`, `description`, `status` (`active`\|`suggested`\|`rejected`), `created_by` (nullable FK `profiles`), `created_at`                                                                                                                                                                                                    |
| `items`                | One pick-able thing within a category          | `id`, `category_id` FK, `name`, `image_url`, `blurb`, `created_at`                                                                                                                                                                                                                                                                        |
| `lineups`              | A user's 6-slot team                           | `id`, `owner_id` FK `profiles`, `category_id` FK, `name`, `explanation` (text), `created_at`, `updated_at`                                                                                                                                                                                                                                |
| `lineup_slots`         | One filled slot in a lineup                    | `id`, `lineup_id` FK, `position` (enum `PG`\|`SG`\|`SF`\|`PF`\|`C`\|`SIXTH_MAN`), `item_id` FK `items`; unique(`lineup_id`, `position`)                                                                                                                                                                                                   |
| `tournaments`          | Unified competition entity (1v1s and brackets) | `id`, `creator_id` FK `profiles`, `name`, `mode` (`single_category`\|`open`), `category_id` (nullable; required if `mode = single_category`), `bracket_size` (int, power of 2), `signup_deadline` (timestamptz), `round_duration` (interval), `status` (`draft`\|`open`\|`in_progress`\|`complete`), `invite_code` (unique), `created_at` |
| `tournament_entries`   | A lineup's entry into a tournament             | `id`, `tournament_id` FK, `lineup_id` FK, `seed` (int), `joined_at`; unique(`tournament_id`, `lineup_id`)                                                                                                                                                                                                                                 |
| `matches`              | One bracket cell                               | `id`, `tournament_id` FK, `round` (int), `slot_in_round` (int), `entry_a_id` FK `tournament_entries` (nullable = bye), `entry_b_id` FK (nullable = bye), `winner_entry_id` FK (nullable until resolved), `voting_opens_at`, `voting_closes_at`, `status` (`pending`\|`open`\|`closed`), `created_at`                                      |
| `votes`                | One vote on a match                            | `id`, `match_id` FK, `voter_id` (nullable FK `profiles`), `voter_fingerprint` (nullable text, anon dedup), `choice_entry_id` FK `tournament_entries`, `created_at`; unique partial constraints on (`match_id`, `voter_id`) and (`match_id`, `voter_fingerprint`)                                                                          |
| `comments`             | Forum/comment thread entry                     | `id`, `tournament_id` FK, `author_id` FK `profiles`, `body`, `created_at`                                                                                                                                                                                                                                                                 |
| `category_suggestions` | User-submitted category ideas                  | `id`, `suggested_by` (nullable FK `profiles`), `name`, `description`, `status` (`pending`\|`approved`\|`rejected`), `created_at`                                                                                                                                                                                                          |

**Row-Level Security:** public read on `categories`/`items`/`lineups`/`tournaments`/`matches`/vote-aggregates/`comments`; writes gated to the resource owner or an authenticated user as appropriate. Vote inserts do **not** go through direct table writes — they go through a `SECURITY DEFINER` Postgres function that validates the fingerprint/voter dedup server-side, so a client can't bypass the one-vote-per-match rule by crafting its own insert.

## Core Loop

1. Browse categories → build a lineup (fill PG/SG/SF/PF/C/6th Man from one category's items) + write an explanation → save → public lineup detail page.
2. Either:
   - **Quick Matchup** — pick your lineup + an opponent lineup, no mode/bracket settings shown. Creates a `bracket_size = 2` tournament under the hood.
   - **Tournament** — set bracket size, mode, signup deadline, round duration → get an invite link → entrants join → auto-start at the deadline (or manual early start) with byes filling empty slots → bracket seeded.
3. Each open match is votable (anonymous or signed-in, one vote, locked in) until its `voting_closes_at`. A scheduled job closes expired matches, resolves ties by seed, advances winners, and opens the next round.
4. Every tournament/matchup page has a comment thread (posting requires an account).
5. A "suggest a category" form feeds `category_suggestions`; an admin reviews (internal page or documented Supabase Studio workflow for MVP) and promotes approved suggestions into `categories`.

## Tech Stack

- **Next.js** (App Router) + TypeScript, **Tailwind CSS** + **shadcn/ui** — styled for a 90s/2000s minimal aesthetic (chunky borders, web-safe-adjacent palette, no gradients/glassmorphism) in the Phase 6 UI pass
- **Supabase**: Postgres + Auth (email/password + Google OAuth) + Storage (item images) + RLS; Realtime deferred post-MVP
- **Vercel** hosting + Vercel Cron (round auto-close job) + Vercel Analytics — actual deployment happens at launch, not before (explicit instruction from the project owner)
- **Google AdSense** for monetization
- **GPL-3.0** license
- **Test-driven development throughout:** Vitest + React Testing Library for units/logic (bracket seeding, tie-break, vote dedup), written before implementation
- **Agent-drivable UI testing:** Playwright as the persisted, CI-run e2e suite covering critical flows (create lineup, create Quick Matchup, vote, create tournament end-to-end including byes/ties, comment). During development, changes are verified interactively against the actual running app (via Playwright or browser automation tools), not just by running the automated suite
- **GitHub Actions CI:** lint, typecheck, unit tests, Playwright e2e, build — required to pass before merge

## Repo Conventions

- Trunk-based: protected `main`, short-lived branches (`feat/…`, `fix/…`, `chore/…`)
- [Conventional Commits](https://www.conventionalcommits.org/) (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`)
- Branch protection: PR + green CI required to merge
- ESLint + Prettier, Husky + lint-staged pre-commit, TypeScript strict mode
- Issue templates (bug/feature), PR template, `CODE_OF_CONDUCT.md` (Contributor Covenant), `CONTRIBUTING.md` (states the TDD expectation), `SECURITY.md`
- `docs/superpowers/specs/` for design docs/ADRs (this document lives here)

## MVP Category Seed List

Four launch categories, per the project owner's own examples: **NBA Players, Sitcom Characters, Politicians, Inanimate Objects** — roughly 20–30 items each for real lineup variety. Actual item lists are drafted during Phase 1 seeding work and require the project owner's review/approval before going live.

## Phased MVP Roadmap

**Phase 0 — Repo & Foundations** ✅ _(this document's companion work)_
`git init`, `.gitignore`, `LICENSE` (GPL-3.0), `README`, `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, issue/PR templates, GitHub repo created and pushed, branch protection on `main`, GitHub Actions CI (lint/typecheck/unit/e2e/build). Next.js + TS scaffold, Tailwind + shadcn/ui, ESLint/Prettier, Husky/lint-staged, Vitest + Playwright harness with a passing smoke test of each.

**Phase 1 — Core Data Model & Auth**
Supabase migrations for all tables above + RLS policies + the `SECURITY DEFINER` vote-insert function. Supabase Auth wired (email + Google OAuth). Seed script + draft item lists for the 4 launch categories (project owner reviews/approves content).

**Phase 2 — Lineup Builder**
Browse categories/items, build a 6-slot lineup with explanation, save/edit own lineups, public lineup detail page. Unit tests first (slot validation, one-category rule) then UI, verified interactively against the running app.

**Phase 3 — Quick Matchup & Voting**
Quick Matchup creation (pick 2 lineups → `bracket_size=2` tournament), public vote page, anonymous + signed-in locked-in voting via the security-definer function, results view, comments on the matchup.

**Phase 4 — Tournaments & Brackets**
Tournament creation (bracket size, mode, signup deadline, round duration), invite link/code, entrants join, bye-filling and seeding at the signup deadline/manual start, bracket visualization, a Vercel Cron job to auto-close expired rounds/resolve ties/advance winners/open the next round, tournament comments.

**Phase 5 — Category Suggestions & Light Admin**
"Suggest a category" form → `category_suggestions`, minimal internal admin review page (or a documented Supabase Studio workflow) to approve/seed into `categories`.

**Phase 6 — Monetization & Launch Polish**
Google AdSense integration + `ads.txt`, full 90s/2000s UI pass, responsive/accessibility pass, SEO/OG tags, production deploy on Vercel (first real deploy — deliberately held until this phase), open-source repo polish (README, CONTRIBUTING clarity) for external contributors.

**Phase 7 — Explicitly post-MVP (not built now):** Supabase Realtime live vote/bracket updates, in-app invite + notifications system, user profiles/follow, ranking/Elo across matchups, more categories, moderation tooling, mobile app.

## Verification

- **Phase 0:** CI pipeline runs green on a trivial PR.
- **Phase 1:** migrations apply cleanly to a fresh Supabase project; RLS policies verified with both authenticated and anonymous Supabase client calls.
- **Phase 2–5:** each phase ships with unit tests written first (TDD) and a Playwright e2e test for its critical user flow, run headless in CI and interactively against the running dev app before the phase is marked done.
- **Phase 6:** Lighthouse pass (performance/accessibility/SEO), manual AdSense placement check, full click-through of create-lineup → create-tournament → vote → comment on a deployed preview.
