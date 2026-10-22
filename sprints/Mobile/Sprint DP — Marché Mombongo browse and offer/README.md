# Sprint DP — Marché Mombongo: browse and offer

> **Retroactive doc** (written 2026-09-01). This sprint shipped without its
> own spec doc — the only record of it was a `roleHomeConfig.ts` comment
> ("Sprint DP (2026-09-01)"). Recorded here for parity with every other
> sprint, after the fact, once discovered during a full sprint-compliance
> audit. **Status: Done**, including test coverage added the same day this
> doc was written (see "Test coverage" below).

## Objective

Let ADMIN act as a buyer on Mombongo's own farmer marketplace — a
different relationship from every other MOB-07–MOB-10 sprint, which are
all about Mombongo processing a payment *AROM* owes a producer. Here,
AROM browses listings farmers have published on Mombongo (any commodity,
not just pineapple) and submits an offer to buy. If a farmer accepts,
the resulting sale becomes a `harvestInvoices` doc (Mombongo's own
`invoiceIssued` webhook) that AROM then pays — that payment flow is
already covered by MOB-07's webhook infrastructure and the merged
"Factures" list (MOB-08's revision), not duplicated here.

## What shipped

- `AROM-Mobile/src/app/harvest-listings.tsx` — ADMIN-only screen (two
  views in one route, mirroring `quality-control-new.tsx`'s
  wizard-in-one-screen pattern): a product picker (discovered live from
  Mombongo's own published listings via a wide `limit: 200` query — not
  a hardcoded commodity list, since what Mombongo carries can change
  without AROM knowing), with `ananas` pinned first and pre-selected
  (AROM's own business) but every other commodity actually listed still
  selectable; then a per-product browser with a province filter and
  pull-to-refresh. Deliberately **not** synced through MOB-02's offline
  pull-cache like every other collection this app reads — a listing's
  whole point is "what's available right now" on a marketplace AROM
  doesn't control, so a stale cached copy risks offering on a listing
  that's no longer active. Always a live fetch.
- `AROM-Mobile/src/app/harvest-listing/[id]/offer.tsx` — the offer form.
  Carries the tapped listing's fields forward as route params (no second
  fetch — a listing has no local doc to look up, it only ever lives on
  Mombongo's side) and lets the admin propose a quantity (capped at what
  the listing has) and price, plus an optional message to the seller.
  Submits via `submitHarvestOffer`, shows a real accepted/rejected/error
  result — never a fabricated success.
- `AROM-Mobile/src/features/harvest/mombongoHarvestGateway.ts` —
  `fetchHarvestListings`, `submitHarvestOffer`, `startHarvestInvoiceCheckout`.
  Same shape as `mombongoProducerPaymentGateway.ts` (MOB-09): calls
  AROM-Production's server routes with the current admin's own Firebase
  ID token, never talks to Mombongo directly, never sees a partner
  secret.
- `AROM-Production/src/routes/api/mombongo/{create-offer,create-harvest-checkout,listings}.ts`
  and `AROM-Production/src/lib/payments/mombongoHarvest.ts` — the
  server-side proxy routes and signing logic, same pattern as MOB-07's
  invoice/checkout routes.
- Entry point: ADMIN home's "Marché Mombongo" action
  (`roleHomeConfig.ts`), routing to `/harvest-listings`.
- No AROM branding of Mombongo anywhere in the mobile UI, matching every
  other Mombongo-adjacent sprint's own hard requirement.

## Test coverage

Added retroactively: `__tests__/HarvestListingsScreen.test.tsx` (product
discovery from real listings with ananas pinned first, province
filtering re-queries the gateway, empty/error states, non-ADMIN
redirect, row tap navigates to the offer screen with the right params)
and `__tests__/HarvestOfferScreen.test.tsx` (pre-fill from route params,
over-quantity validation disables submit, accepted/rejected result
states, non-ADMIN redirect) — bringing this flow to the same coverage
bar as every other feature in the app.

## Known gaps

- No sprint-time acceptance-criteria doc existed to check this against —
  this README is a description of what was built, not a spec that
  preceded it. Future changes to this flow should get a proper spec
  doc up front, matching every other sprint's convention.
- Same non-goal as every other Mombongo integration in this app: no
  live end-to-end verification against Mombongo's production
  environment is claimed here — see MOB-07/MOB-10's own notes on what
  "test mode" verification still means before a real go-live.
