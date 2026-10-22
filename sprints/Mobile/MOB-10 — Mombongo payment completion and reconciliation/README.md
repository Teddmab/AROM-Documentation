# MOB-10 — Completion and go-live (revised — the open reconciliation question is answered)

> **Status: Mostly done** (verified 2026-09-01). Every buildable
> acceptance criterion is met: the stuck-invoice banner
> (`STUCK_THRESHOLD_HOURS = 2` in `producer-invoice/[id]/index.tsx`) and
> the [runbook.md](../../../runbook.md#mombongo-payments-test-mode-and-go-live)
> "test mode and go-live" section both exist. What's left — a live
> end-to-end `testMode` run against Mombongo's real environment and a
> real webhook-retry recovery test — is an operational verification that
> cannot be confirmed from a static code audit; the runbook itself notes
> mobile money hadn't been confirmed completing live as of its writing.
> Re-run this checklist once MOB-07's Cloudflare secrets are deployed.

## Revision note (2026-08-31)

The original open question — "what does a paid Mombongo invoice mean for
AROM's financial records, does it need a new bridge like order→ventes?"
— **is answered by the architecture correction in MOB-07/09**: this
integration processes payment for an **existing** `producerInvoices`
doc, not a new external-invoice type. There is no new financial-record
question to resolve — a Mombongo-confirmed payment is exactly the same
event a manually-confirmed producer payment always was:
`producerInvoices.statut` transitions to `payee`. Whatever downstream
reporting already reads that field (if anything does today — confirm,
don't assume) continues to work unchanged. This sprint shrinks from
"design a new reconciliation bridge" to "verify the existing one, if any,
still holds."

## Objective

Verify the full chain end-to-end against Mombongo's real (`testMode`)
environment, and close out the go-live checklist their contract's §6
already laid out.

## Technical approach

1. **Mombongo's §6 checklist, AROM's half**:
   - Send Mombongo AROM's webhook receiver URL (MOB-07) once deployed.
   - Run the "first live invoice, in test mode" together — a real
     `testMode: true` invoice through the full chain, not a local fixture
     this time.
2. **Retry/backoff verification** — Mombongo's side already retries their
   outbound webhook 3× with backoff and may manually re-trigger; confirm
   MOB-07's receiver is genuinely idempotent under a real repeated
   delivery, not just in a unit test.
3. **Failure visibility** — if a checkout is created but no webhook ever
   arrives (network issue, Mombongo-side failure), an admin needs to see
   that an invoice is stuck, not just a spinner forever. Add a simple
   "stuck if `checkout_created` for longer than N hours" indicator on the
   existing priority/detail screens (MOB-08) — no new collection, just a
   derived state off existing fields.
4. **Flip `testMode` off** — the actual go-live moment, gated behind an
   explicit decision (env flag flip), not a code change. Document the
   exact steps in `runbook.md` alongside how `PAWAPAY_...` credentials
   are already documented there.

## Acceptance criteria

- A real `testMode` invoice, created by MOB-07/09 against Mombongo's
  actual (non-local) environment, completes end-to-end: checkout created
  → payment completed on Mombongo's side → webhook received by AROM →
  `producerInvoices.statut` becomes `payee` → the mobile wizard's result
  screen reflects it.
- A deliberately-broken receiver (simulate Mombongo's webhook hitting a
  temporarily-down AROM endpoint) correctly recovers once AROM is back up
  and Mombongo retries.
- A stuck-invoice state is visible to an admin, not silent.
- `runbook.md` documents the go-live flip (test → real) as an explicit,
  reversible step.
- `bun run typecheck`, `bun run lint`, `bun run test` (AROM-Mobile and
  AROM-Production) and the Firestore rules suite (AROM-Backend) all pass.

## Out of scope

- Any new payment method beyond card/mobile money (bank transfer is
  still Mombongo's `501` to resolve, not AROM's).
- A production-facing scenario/testing UI — MOB-06B's dev-only concept
  isn't carried forward here; `testMode` on the real gateway is the
  simulation mechanism now.

## Definition of done for the whole MOB-07–MOB-10 initiative

An AROM admin can review a producer invoice, tap "Payer la facture,"
complete real payment via Mombongo (card or mobile money), and see it
marked `payee` — driven by Mombongo's real webhook, not a mock timer.
