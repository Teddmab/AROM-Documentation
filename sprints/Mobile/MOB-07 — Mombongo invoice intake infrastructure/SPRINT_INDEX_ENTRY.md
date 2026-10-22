## MOB-07 — Mombongo payment gateway proxy

**Status: Mostly done** (verified 2026-09-01) — code-complete and
provisioned; Cloudflare Worker secrets deploy and AROM-Production test
coverage remain, see this folder's README.

Revised 2026-08-31 against Mombongo's real, locally-verified API contract:
AROM calls Mombongo's `createExternalInvoice`/`createExternalInvoiceCheckout`
(not the reverse), so this shrinks to a server-side secret-holding proxy
(new AROM-Production TanStack Start routes, PawaPay-precedent pattern) plus
one inbound webhook receiver that flips an existing `producerInvoices` doc
to `payee` via the Admin SDK. No new invoice collection needed.
