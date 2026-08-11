# ADR-0006: Profile & stats

## Status

Accepted — resolved via wayfinder ticket [#7](https://github.com/Ixiandesign/Starting5/issues/7).

## Context

Profiles need a precise definition of "win," where placements live, whether stats are
computed live or maintained, and whether profiles are public at all.

## Decision

- **Lineups shown**: published only (reaffirms [ADR-0002](0002-lineup-structure-and-lifecycle.md))
  — drafts stay private to the owner.
- **Tournament win**: being the champion of a completed tournament. Tracked as an aggregate
  `profiles.tournaments_won` count.
- **Match win**: winning an individual match, independent of whether the tournament as a whole
  was won. Tracked as aggregate `profiles.match_wins` / `profiles.match_losses` counts — shown
  as a won-loss record distinct from tournament championships.
- **Placement**: a per-entry result on `tournament_entries.result` (`champion` or
  `eliminated_round_N`), set when a tournament completes. Not aggregated — queried live per
  profile (cheap, indexed) rather than pre-computed, since it varies per tournament rather than
  rolling up to a single number.
- **"Categories won in"**: distinct `category_id` values across a user's `champion` entries in
  `single_category` tournaments only. `cross_category` championships count toward
  `tournaments_won` but attribute to no single category's win tally.
- **Computation strategy**: aggregate columns (`tournaments_won`, `match_wins`, `match_losses`),
  updated incrementally by the same Vercel Cron sweep that closes matches and completes
  tournaments ([ADR-0004](0004-tournament-lifecycle.md)) — not a separate materialized-view
  refresh job, and not computed at profile read time.
- **Public visibility**: every profile is public — viewable by anonymous visitors and account
  holders alike. No profile privacy setting for MVP, consistent with the map's Out-of-scope
  exclusion of friends-only visibility.

## Consequences

- Profile reads stay cheap (aggregate columns + one indexed query for placements), no read-time
  fan-out over matches/tournaments.
- Stat updates ride the existing cron sweep instead of introducing a second scheduled job.
- Cross-category wins not attributing to a specific category could look like an undercount to
  users who won a cross-category bracket — acceptable trade-off given cross-category wins don't
  map cleanly to one category anyway.

## References

- Builds on [ADR-0002](0002-lineup-structure-and-lifecycle.md) (published-only visibility) and
  [ADR-0004](0004-tournament-lifecycle.md) (the cron sweep this rides on).
