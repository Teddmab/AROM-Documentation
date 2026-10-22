## MOB-10 — Completion and go-live

**Status: Mostly done** (verified 2026-09-01) — buildable criteria
(stuck-invoice banner, runbook) met; live `testMode` end-to-end
verification remains operational, not code, see this folder's README.

Revised 2026-08-31: the original open reconciliation question is answered
— a Mombongo-confirmed payment is just the existing `producerInvoices`
`payee` transition, no new financial bridge needed. Shrinks to end-to-end
verification against Mombongo's real `testMode` environment, retry/
idempotency verification, stuck-invoice visibility, and the explicit
test→real go-live flip.
