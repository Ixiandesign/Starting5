# ADR-0007: Main feed

## Status

Accepted — resolved via wayfinder ticket [#8](https://github.com/Ixiandesign/Starting5/issues/8).

## Context

The main feed is the primary discovery surface for public tournaments and open votes. Needs
exact inclusion criteria, sort, filters, pagination, and a logged-out-vs-logged-in answer.

## Decision

- **Two card types**, both scoped to `visibility = 'public'` tournaments only (invite-only
  tournaments never appear, matching [ADR-0004](0004-tournament-lifecycle.md)):
  - **Needs entrants**: tournaments with `status = 'signup_open'`.
  - **Open for voting**: any currently-open match (`status = 'open'`, `closes_at` in the future)
    belonging to a public tournament — including bracket-size-2 quick 1v1s.
- **Sort**: default is **closing-soonest** (signup deadline for entrant cards, `closes_at` for
  match cards) — most actionable framing for a time-boxed voting feed. Secondary options:
  newest, most-active (vote/entrant count so far, descending).
- **Filters**: by category (the active category set) and by card type (needs-entrants vs.
  open-for-voting). No status filter beyond that — the feed only ever shows open items by
  definition; closed/completed items live on profiles and tournament pages, not the feed.
- **Pagination**: cursor-based (keyset, on the active sort's underlying column), page size 24,
  advanced via a "Load more" button — not infinite scroll, consistent with the map's
  no-realtime/no-disguised-polling stance.
- **Logged-out vs. logged-in**: identical feed content for both — no personalization for MVP
  (no "tournaments your friends are in," consistent with the map's Out-of-scope on
  friends-only filtering). Anonymous visitors can vote directly from feed matches
  ([ADR-0001](0001-auth-session-and-profile-creation.md)); they're only prompted to sign in when
  attempting to enter a tournament.

## Consequences

- Feed queries stay simple (two filtered/sorted/paginated lists over `tournaments`/`matches`,
  scoped by `visibility`), no personalization join against a friends/follow graph.
- "Load more" avoids infinite-scroll's implicit continuous-refresh feel, matching the map's
  explicit no-realtime scope.

## References

- Builds on [ADR-0004](0004-tournament-lifecycle.md) (tournament status/visibility) and
  [ADR-0003](0003-voting-and-matchup-mechanics.md) (match status).
