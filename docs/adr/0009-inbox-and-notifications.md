# ADR-0009: Inbox & notifications

## Status

Accepted — resolved via wayfinder ticket [#9](https://github.com/Ixiandesign/Starting5/issues/9).

## Context

The map calls for an inbox with no realtime delivery. This ticket pins down which events land
in it, read/unread tracking, delivery mechanism, and whether entries support inline actions.

## Decision

- **Event types** — four, one more granular than the ticket's proposed list:
  - `tournament_invite` — a friend invited you to an invite-only tournament. This introduces a
    small addition to [ADR-0004](0004-tournament-lifecycle.md)/[ADR-0008](0008-friend-system.md):
    a `tournament_invites` row (`tournament_id`, `invited_user_id`, `invited_by`) as a second,
    optional discovery path alongside the raw invite-code link — sending one from your friend
    list doesn't change ADR-0004's rule that the link alone is sufficient, it just also notifies
    a specific friend and grants them entry access. Flagged below as an extension worth a look.
  - `match_result` — a match you entered closed (win or loss).
  - `tournament_result` — a tournament you entered completed (your final placement).
  - `friend_request` — someone sent you a friend request.
  - Comment-reply notifications were considered and **excluded** for MVP — the ticket's proposed
    list didn't include them and the map doesn't call for them; noting the gap since the ticket
    asked to confirm completeness.
- **Read/unread**: `inbox_items` (`user_id`, `type`, a payload reference, `created_at`,
  `read_at` nullable). Unread = `read_at IS NULL`. Marked read individually on click-through,
  plus a bulk "mark all read."
- **Delivery**: no polling job. Each event-producing action inserts its own `inbox_items` row
  synchronously — the cron sweep on match/tournament close ([ADR-0004](0004-tournament-lifecycle.md)),
  the friend-request RPC, the tournament-invite action. The client fetches inbox contents fresh
  on page load/navigation only — no client-side polling interval, no websockets.
- **Inline actions**: `tournament_invite` and `friend_request` entries get inline accept/decline
  buttons calling the same RPC/action their dedicated pages use — no navigation required.
  `match_result` and `tournament_result` entries are read-only informational links to the
  relevant page; there's nothing to accept or decline.

## Consequences

- Inbox writes piggyback on existing event-producing code paths — no separate notification
  worker or polling job to maintain.
- The `tournament_invites` addition gives invite-only tournaments a second, friend-targeted
  discovery path beyond the raw link — a small scope extension flagged for confirmation.
- Excluding comment-reply notifications keeps the event list to four, but forum engagement gets
  no inbox nudge for MVP — flagged as a possible gap.

## References

- Extends [ADR-0004](0004-tournament-lifecycle.md) (invite-only discovery) and
  [ADR-0008](0008-friend-system.md) (friend list) with the `tournament_invites` mechanism.
