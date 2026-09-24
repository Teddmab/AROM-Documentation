# Sprint 02 Cross-device freshness

Read `docs/completion-plan/README.md` first. Keep the cache-first architecture and mutation queue.

Add one centralized refresh coordinator that:

- renders cached data immediately;
- refreshes authorized read plans when the application returns to the foreground;
- refreshes ADMIN cross-role screens when they gain focus;
- deduplicates simultaneous refreshes;
- throttles repeated focus events using an existing threshold or one clearly documented constant;
- cancels or ignores obsolete results safely;
- keeps manual refresh as a fallback;
- stores and displays the last successful refresh time.

Support distinct fresh, stale, offline, partial, and failed-refresh states. Do not add listeners to every collection, introduce Redux or React Query, or build a background service.

Tests must prove:

- a record written by another device becomes visible after foreground or focus refresh;
- queued local mutations remain merged exactly once;
- focus storms do not issue duplicate fetches;
- failed refresh keeps usable cached data visible;
- role changes still invalidate unauthorized cached surfaces.

