## MOB-06A — Administrator oversight and producer payables foundation

**Status: Done** (verified 2026-09-01) — see this folder's README for evidence.

Replaces the thin ADMIN home with a cache-first, actionable overview of receptions, production, the real `qualityControls` workflow, raw-material `stockMP`, sales, tasks, staff activity and synchronization health, while introducing a provider-neutral AROM foundation for producer invoice review. The sprint must not invent finished-goods stock or allow resolution of quarantined lots while MOB-05C is deferred. Invoice ingestion and payment execution remain capability-gated until a real backend contract exists; the app never exposes Mombongo branding, fabricates invoices, or simulates payment success. Shared AppHeader/BackHeader, `Mode Administrateur` badge and BottomNav are mandatory across every route.
