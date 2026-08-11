# ADR-0005: Comments & moderation

## Status

Accepted — resolved via wayfinder ticket [#6](https://github.com/Ixiandesign/Starting5/issues/6).

## Context

Every matchup — standalone or part of a tournament — needs threaded discussion. Since
[ADR-0003](0003-voting-and-matchup-mechanics.md) made a standalone "quick 1v1" a
`bracket_size = 2` tournament rather than a separate concept, comment scoping can build on that
same unification.

## Decision

- **Thread scope**: comments attach only to a tournament (`comments.tournament_id`, required) —
  never to an individual match/round. Because a quick 1v1 is itself a tournament (ADR-0003),
  every matchup already has exactly one natural thread; per-round threads inside a bracket would
  fragment discussion without adding value.
- **Nesting**: max depth 3 (top-level → reply → reply-to-reply); deeper replies flatten visually
  to level 3 but remain postable and readable.
- **Comment voting**: none for MVP — only lineups get votes. Sort order is `new` (default) or
  `oldest`; no `best`/`top` sort since there's no vote signal to rank by. Flagged below since
  "Reddit-style" might imply vote-based ranking was expected.
- **Edit/delete**: the author can edit anytime as long as the comment has no replies yet; once
  replied-to, it locks (mirrors the lock-on-use pattern already used for lineups and votes). The
  author can always delete; deleting a comment with replies soft-deletes it (content replaced
  with "[deleted]", replies remain) rather than orphaning the thread.
- **Moderation**: a report button only. Reported comments are flagged for admin review (a
  `comment_reports` table); no automated hide threshold for MVP. This resolves the map's
  "Not yet specified" item on moderation depth.
- **Account requirement**: reaffirms the map's existing standing preference — commenting
  requires an account, not re-litigated here.

## Consequences

- Reusing the tournament-as-thread-owner model means no new "matchable" polymorphism — comments
  have exactly one foreign key, not an either/or.
- No comment voting keeps the vote/dedup machinery scoped to lineups only, at the cost of no
  community-driven "best comment" surfacing — flagged for confirmation.
- Report-only moderation is minimal but matches MVP scope; if abuse becomes a problem, the
  `comment_reports` table is the seam to build an admin queue against later.

## References

- Builds on [ADR-0003](0003-voting-and-matchup-mechanics.md)'s unification of quick-1v1 and
  tournament.
