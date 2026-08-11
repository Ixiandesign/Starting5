# ADR-0008: Friend system

## Status

Accepted — resolved via wayfinder ticket [#10](https://github.com/Ixiandesign/Starting5/issues/10).

## Context

The map scopes the friend system to a minimal connection list for MVP — no visibility/filtering
features yet. This ticket pins down the request flow, where the list surfaces, and whether it
does anything functional beyond existing.

## Decision

- **One table**: `friend_requests` (`requester_id`, `addressee_id`, `status`: `pending` |
  `accepted`). No separate `friendships` table — an accepted row *is* the friendship; membership
  reads as `WHERE (requester_id = me OR addressee_id = me) AND status = 'accepted'`.
- **Flow**: sending checks both directions for an existing `pending`/`accepted` row before
  insert (no duplicate or crossed requests). Accept flips `status` to `accepted`. Decline
  **deletes** the row rather than keeping a permanent `declined` state — the requester can send
  a fresh request later; no cooldown mechanism for MVP. Unfriending (either party) deletes the
  `accepted` row.
- **Where surfaced**: on the viewer's own account area only (a "Friends" section with your list
  plus pending incoming/outgoing requests) — not shown on other users' public profile pages.
  Keeps [ADR-0006](0006-profile-and-stats.md)'s "every profile is public" scoped to stats/
  lineups, not the social graph.
- **Functionality**: purely social for MVP. Friendship does **not** feed into invite-only
  tournament enrollment — that stays link-only per [ADR-0004](0004-tournament-lifecycle.md), no
  friend-gated enrollment path. Flagged below since this declines an explicit option the ticket
  raised.

## Consequences

- No social-graph exposure on public profiles, keeping the friend system additive rather than
  reshaping the already-decided public-profile model.
- Declining reopens the door to future requests without extra state — simple, but means a
  declined requester isn't blocked from immediately retrying (accepted as fine for MVP; real
  anti-harassment tooling is out of scope per the map).
- Friendship staying purely social means invite-only tournaments gain no friend-list shortcut for
  MVP — flagged in case that was an expected use case.

## References

- Reaffirms the map's Out-of-scope note: no friends-only visibility/filtering yet.
