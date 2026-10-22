# MOB-08 — Extend the existing producer-invoice screens (revised — no new screens)

> **Status: Done** (verified 2026-09-01). `producer-invoice/[id]/index.tsx`
> shows the Mombongo checkout-in-progress state and `TestModeBanner`;
> `ProducerInvoiceDetailScreen.test.tsx` covers the extension. Matches
> this sprint's deliberately small scope exactly.

## Revision note (2026-08-31)

This sprint originally proposed a brand-new `mombongoInvoices` list and
detail screen. That's no longer needed: MOB-07's revision confirmed the
real integration target is AROM's **existing** producer-invoice flow —
`app/producer-invoices.tsx` (list) and `app/producer-invoice/[id]/index.tsx`
(detail), both already built in MOB-06B, already guided-contract-
compliant, already showing exactly the fields (producer, amount, due
date, status) a Mombongo-processed invoice needs. This sprint shrinks to
the small, real additions those screens need to reflect a Mombongo
payment in flight — not a rebuild.

## Objective

Make the existing producer-invoice detail screen show Mombongo-specific
state (which checkout method is in progress, a `testMode` indicator
during the simulation phase) without changing its guided-screen
structure, and make sure MOB-07's new `producerInvoices` fields
(`mombongoInvoiceId`, `mombongoCheckout`) sync correctly through the
existing offline-sync pattern.

## Technical approach

1. **Sync**: `producerInvoices` already syncs for ADMIN (MOB-04/06A) —
   confirm the two new fields from MOB-07 round-trip through the existing
   repository/cache layer without any change to the sync plan itself
   (they're just new optional fields on an existing type).
2. **`app/producer-invoice/[id]/index.tsx`**: when `mombongoCheckout` is
   present and not yet resolved, show a small status line under the
   existing `BusinessStatusBadge` (e.g. "Paiement en cours via Mombongo
   — {method}") — reuse `StatusBanner`'s `pending` kind, don't invent a
   new banner component.
3. **`DemoDataBanner`-equivalent for `testMode`**: while
   `EXPO_PUBLIC_ENABLE_MOCK_PAYMENTS`-equivalent test flag is on and
   Mombongo's own `testMode` flag is stamped on the invoice, show the
   existing `DemoDataBanner` component as-is (MOB-06B already built
   exact required copy for "this is a simulation" — reuse it verbatim
   rather than writing new copy for what is, functionally, the same
   warning).
4. **No new priority-item type needed** — `adminPriorities.ts` already
   surfaces an approved-but-unpaid invoice; no change required there.

## Acceptance criteria

- A `producerInvoices` doc with `mombongoCheckout` set displays its
  in-progress state on the existing detail screen, without breaking any
  existing MOB-06B test assertion (`InvoiceComparisonCard`, `WhatNextCard`,
  status badges all unchanged).
- `testMode` invoices show `DemoDataBanner`; real ones don't.
- `bun run typecheck`, `bun run lint`, `bun run test` pass, with the
  existing `ProducerInvoiceDetailScreen.test.tsx` extended (not
  replaced) for the two new states above.

## Out of scope

- Payment execution itself (MOB-09).
- Any new route or screen — if this sprint starts requiring one, that's
  a signal the MOB-07 data model needs rethinking, not that a new screen
  is the fix.
