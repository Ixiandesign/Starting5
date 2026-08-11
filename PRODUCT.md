# Product

<!-- impeccable:product-schema 1 -->

## Platform

web

## Stack

Next.js (App Router) + TypeScript, Tailwind CSS + shadcn/ui, Supabase (Postgres + Auth),
Vercel hosting. Fixed by the wayfinder map ([issue #1](https://github.com/Ixiandesign/Starting5/issues/1))
— not a greenfield choice made here.

## Users

Casual sports/pop-culture fans who enjoy debate and voting: people who'd argue "who's really
the GOAT starting five" for any category, not just basketball. Two situations: signed-in
creators building and entering lineups/tournaments, and anonymous browsers who read and vote
without an account.

## Product Purpose

Build a "Starting 5+1" lineup — 6 fixed positions (`PG`/`SG`/`SF`/`PF`/`C`/`SIXTH_MAN`) — from
any category (NBA players, sitcom characters, politicians, inanimate objects), then have the
community vote on 1v1 matchups and single-elimination tournaments between lineups.

## Positioning

Bracket-voting apps and fantasy roster builders both exist separately; Starting5's mechanism is
forcing *every* category into the same fixed basketball-lineup shape regardless of subject —
the "Starting 5+1" conceit is the thing a neighboring product couldn't copy without becoming a
different app. (See [CONTEXT.md](CONTEXT.md)'s Slot/Lineup glossary entries.)

## Operating Context

Web app on Supabase Auth (email/password + Google OAuth) with anonymous browsing and voting.
Tournament rounds advance via a Vercel Cron sweep, not realtime push — no websockets, no
infinite scroll, refresh/revalidate-on-navigation only. Full mechanics: [CONTEXT.md](CONTEXT.md)
and `docs/adr/0001`–`0009`.

## Capabilities and Constraints

Resolved via the wayfinder grilling process (ADR-0001 through 0009): account vs. anonymous
boundary, lineup lifecycle (draft/published), unified match/quick-1v1 model, tournament
lifecycle (bracket sizes, byes, tie-break, cron advance), tournament-scoped comment threads,
profile stats, main feed, friend system, inbox/notifications. Several product decisions remain
flagged for owner sign-off — see the "Needs owner sign-off" section of
[map issue #1](https://github.com/Ixiandesign/Starting5/issues/1) before treating any of them as
final.

## Brand Commitments

Name: **Starting5**. Core term: **"Starting 5+1"** (never "roster" or "team" as a synonym — see
CONTEXT.md). License: GPL-3.0.

## Evidence on Hand

Draft category seed content exists at [docs/category-seeds-draft.md](docs/category-seeds-draft.md)
(4 launch categories, ~24 items each) — **draft only**, not owner-approved or seeded live (see
wayfinder ticket #12). No screenshots or other demo data exist. This project is still in the
wayfinder spec phase (no application code in the repo). Any imagery or content used in early
visual work is illustrative/synthetic and must be labeled as such, not treated as real product
content.

## Product Principles

- One category, one lineup shape — the six basketball-position slots never vary by category;
  that consistency is the joke and the mechanism, not a limitation to design around.
- Functional over promotional — every current surface is Operate-mode (task completion:
  browse, build, vote, comment), not a marketing page. No landing/persuade surface exists yet.
- No realtime, no infinite scroll — state changes on explicit navigation/refresh only; the UI
  should never imply live-updating data it doesn't have.
- Anonymous-friendly by default — browsing and voting must never visually gate behind an
  account; only creation actions do.
