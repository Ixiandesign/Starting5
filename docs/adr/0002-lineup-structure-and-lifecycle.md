# ADR-0002: Lineup structure & lifecycle

## Status

Accepted — resolved via wayfinder ticket [#3](https://github.com/Ixiandesign/Starting5/issues/3).

## Context

Every category (NBA players, sitcom characters, politicians, inanimate objects, ...) needs to
produce a "Starting 5+1" lineup that can be voted on in matchups and tournaments.

## Decision

- **Slots are fixed and category-independent**: every lineup has exactly 6 slots — `PG`, `SG`,
  `SF`, `PF`, `C`, `SIXTH_MAN` — regardless of category. This is the app's core conceit (the
  name "Starting5"), not a basketball-only feature.
- **One category per lineup**: `lineups.category_id` is required; all 6 slot picks (`items`)
  must belong to that same category (enforced via FK + check constraint).
- **Explanation**: a single lineup-level `explanation` text field, required, 10-1000 chars,
  shown alongside the lineup wherever it's displayed or voted on. Per-slot rationale was
  considered and deferred — flagged below.
- **Lifecycle**: `status` enum `draft` | `published`.
  - `draft`: private to the owner, fully editable/deletable, not enterable into any tournament
    or matchup, not votable.
  - `published`: one-way transition from draft (no unpublish). Visible in feed/profile,
    votable, editable/deletable by the owner **until** the lineup is referenced by a
    `tournament_entries` or `matches` row — at that point it's locked (immutable, undeletable)
    to protect vote/bracket integrity.

## Consequences

- Answers wayfinder ticket #7 (Profile & Stats)'s open "all lineups or just published?"
  question: profiles show published lineups only; drafts stay private to the owner.
- Locking-on-use mirrors the vote-lock-in pattern already fixed at the map level — one
  consistent integrity story across lineups, votes, and matches.
- A single explanation field is simpler to build and display than per-slot explanations;
  flagged for confirmation in case per-pick reasoning matters more to the product than assumed.

## References

- Reaffirms the `lineup_slots` position enum from the pre-wayfinder Phase-1 plan (git history,
  commit `3b45274`).
