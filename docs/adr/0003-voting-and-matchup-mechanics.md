# ADR-0003: Voting & matchup mechanics

## Status

Accepted — resolved via wayfinder ticket [#4](https://github.com/Ixiandesign/Starting5/issues/4).

## Context

Every vote — standalone or as part of a tournament round — needs one consistent,
abuse-resistant mechanism, and cross-category tournaments need a matchup model that doesn't
require normalizing across categories.

## Decision

- **Unified match model**: a `matches` row is the atomic voting unit — two lineups
  (`lineup_a_id`, `lineup_b_id`), an open/closed `status`, `opens_at`/`closes_at`, `winner_id`
  once closed. Both a standalone "quick 1v1" and every tournament round use this same table.
- **Quick 1v1 = bracket-size-2 tournament**: no separate code path. A "quick 1v1" is a
  tournament with `bracket_size = 2` — one match, no further rounds. Keeps ticket #5
  (Tournament mechanics) to a single generalized bracket engine instead of two systems.
- **Cross-category matchups**: mechanically identical to same-category — a match just pairs two
  lineups and voters compare explanations, no category-specific scoring or normalization
  needed. `tournaments.category_mode` (`single_category` | `cross_category`) drives whether
  `category_id` is required on the tournament; `matches` itself doesn't care about category.
- **Vote casting**: writes only through a `SECURITY DEFINER` `cast_vote(match_id,
  choice_lineup_id)` function — never a direct client insert. Validates the match is currently
  open and that `choice_lineup_id` is one of the match's two entrants.
- **Dedup**: partial unique index on `(match_id, voter_id)` for signed-in voters, `(match_id,
  fingerprint)` for anonymous voters. `fingerprint` = a random token in a signed, httpOnly
  cookie set on first visit — not canvas/device fingerprinting. Sufficient for MVP; a cleared
  cookie can bypass it, an accepted trade-off (flagged below).
- **Lock-in**: enforced structurally, not just by app logic — there is no update or delete path
  for `votes` at all, only the insert-only RPC. A cast vote cannot change.
- **Live results**: vote counts are visible while a match is open, refreshed on page
  load/revalidate — consistent with the map's "no realtime push" scope (visibility itself isn't
  restricted, only live-push is). Not a blind-until-close model.

## Consequences

- One bracket/match engine serves quick 1v1s and every tournament round — no duplicated logic
  between "quick vote" and "tournament round" code paths.
- Cross-category tournaments need no special-cased comparison logic.
- Cookie-based dedup is bypassable by clearing cookies; acceptable for MVP since the map already
  defers stronger anti-abuse, flagged for confirmation.
- Visible-while-open (not blind-until-close) vote counts is a product/suspense call some owners
  might want reversed — flagged for confirmation.

## References

- Reaffirms the `cast_vote()` `SECURITY DEFINER` pattern and partial-unique-index dedup from the
  pre-wayfinder Phase-1 plan (git history, commit `3b45274`).
