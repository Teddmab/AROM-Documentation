# Sprint 07 Administrator experience

Read `docs/completion-plan/README.md` first.

Keep four distinct purposes:

- Home: what needs attention now.
- Activity Journal: what happened.
- Operational Summary: what changed during the selected period.
- Priorities: the complete deduplicated action queue.

Replace `Suivi des opérations` on Home with `Voir l’activité récente`, routed to `/activity`. Remove navigation entry points to `/admin-operations`; retain only a temporary redirect if deep-link compatibility is required.

Rules:

- never infer health from a zero count;
- never claim freshness when cached data is stale;
- never invent an author, threshold, comparison, or metric;
- count each lot using its latest effective quality control;
- do not count a partner order and linked sale twice;
- aggregate cached ADMIN data in one memoized pass;
- priorities use a stable entity-type, entity-ID, and priority-type key and disappear when the underlying condition resolves.

Test aggregation, routing, deduplication, stale and partial data, period boundaries, accessibility, and empty states.

