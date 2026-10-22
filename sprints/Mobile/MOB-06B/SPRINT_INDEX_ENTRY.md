## MOB-06B — Guided Administrator redesign and mock producer payments

**Status: Done** (verified 2026-09-01) — both workstreams shipped; a
manual device visual pass and the producer-name join remain, see this
folder's README.

Visually remediates every MOB-06A Administrator dashboard and invoice route for nontechnical users through simpler hierarchy, contextual imagery, plain-language explanations, persistent guidance cards and one dominant action per screen. It also implements an unmistakably development-only mock payment journey behind the existing capability boundary, covering verification, confirmation, processing, success, pending, failure and idempotency without transferring money or mutating real Firestore invoice payment statuses. Exact route-by-route screenshot comparison, shared AppHeader/role badge/BottomNav, small-screen behavior and production lockout are definition-of-done requirements.
