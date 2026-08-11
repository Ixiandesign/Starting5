# ADR-0004: Tournament lifecycle

## Status

Accepted — resolved via wayfinder ticket [#5](https://github.com/Ixiandesign/Starting5/issues/5).

## Context

Tournaments are single-elimination, randomly seeded, creator-configured (per the map's standing
preferences). This ticket pins down bracket sizes, category mode, enrollment, seeding timing,
byes, tie-breaks, and the round-advance mechanism.

## Decision

- **Bracket sizes**: creator picks one of `{2, 4, 8, 16, 32, 64}` (power-of-two, matching the
  pre-wayfinder plan's check constraint). `bracket_size = 2` is the "quick 1v1" case from
  [ADR-0003](0003-voting-and-matchup-mechanics.md).
- **Category mode**: binary, matching ADR-0003 — `single_category` (`category_id` required, all
  entrant lineups must belong to it) or `cross_category` (`category_id` null, entrant lineups
  can come from any published category). No curated "named subset of categories" middle option
  for MVP.
- **Enrollment / visibility**: `tournaments.visibility` = `public` | `invite_only`. Public
  tournaments are listed on the main feed; any account holder can enter a published lineup up to
  `bracket_size`. Invite-only tournaments aren't listed on the feed and are reachable only via a
  unique `invite_code` link — still requires an account to enter, but discovery is the link
  instead of the feed. Not tied to the friend system (map's Out-of-scope excludes
  friends-only filtering).
- **Signup deadline & voting duration**: creator sets both at creation. Signup window: 1 hour to
  30 days out. Per-round voting duration: 1 hour to 7 days, applied uniformly to every round
  (no per-round customization for MVP). Both are adjustable defaults, not load-bearing decisions
  — flagged lightly below.
- **Seeding timing**: triggered by whichever comes first — the signup deadline passing, or the
  creator manually starting the tournament early (requires ≥ 2 entrants). Both paths run the
  same seeding routine.
- **Minimum entrants**: fewer than 2 entrants at trigger time → tournament auto-cancels
  (`status: cancelled`) rather than running with a walkover champion of one.
- **Byes**: entrants are randomly seeded into the bracket's slots; any slots beyond the entrant
  count are byes, randomly distributed (not clustered) by including "empty" as a possible
  outcome in the same shuffle. A first-round match with a bye on one side never opens for
  voting — its human entrant auto-advances at seeding time.
- **Tie-break**: an exact vote-count tie at a round's close resolves immediately via a
  deterministic random tiebreak seeded by `(match_id, closes_at)` — reproducible and auditable,
  not silent chance and no voting extension. Flagged below as a product call worth confirming.
- **Round auto-advance**: a Vercel Cron job hits a route handler every 5 minutes. Each run:
  1. Seeds any tournament whose signup deadline just passed (or was manually early-started) and
     isn't seeded yet — generates round 1 matches per the seeding/bye rule above.
  2. Closes any open match whose `closes_at <= now()` — computes the winner from votes (or the
     tie-break rule, or the bye rule).
  3. If a closed match was the tournament's final round, marks the tournament complete and sets
     the champion; otherwise generates the next round's matches by pairing this round's winners,
     with a fresh `opens_at`/`closes_at` window.

## Consequences

- One cron sweep drives every tournament's seeding and round transitions — no per-tournament
  scheduled jobs to manage.
- Deterministic tie-break avoids both silent randomness and voting-extension complexity, at the
  cost of not giving voters a chance to "settle" a tie themselves — flagged for confirmation.
- Auto-cancel on under-2-entrants avoids a degenerate 1-entrant "championship," at the cost of
  the creator possibly losing a tournament they hoped would fill later — acceptable since they
  can just recreate it.

## References

- Reaffirms the `bracket_size` power-of-two constraint and Vercel Cron round-advance approach
  from the pre-wayfinder Phase-1 plan (git history, commit `3b45274`).
- Builds on [ADR-0003](0003-voting-and-matchup-mechanics.md)'s unified match model and
  category-mode field.
