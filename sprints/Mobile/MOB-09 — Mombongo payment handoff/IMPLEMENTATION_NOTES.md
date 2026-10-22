# MOB-09 — implementation notes (as built, 2026-08-31, simulation phase)

## Scope actually delivered this pass

Only the **AROM-Mobile, `mode: "real"`-shaped simulation** — the part
unblocked without Mombongo's real deploy/credentials. AROM-Production's
server routes, real secrets, and the inbound webhook receiver (MOB-07)
are **not** built — they're blocked on Mombongo's own §6 handoff
(partner id + two secrets, sent once they deploy).

## What's new

- `src/features/invoices/model.ts` — `ProducerInvoice` gained optional
  `mombongoInvoiceId`/`mombongoCheckout` fields (additive, matches what
  MOB-07's future server routes will stamp).
- `src/features/payments/gateway.ts` — `ProducerPaymentGateway.startPayment`
  widened to accept optional `method`/`phone`/`operator`; `PaymentResult`
  gained an optional `checkout: MombongoCheckoutInfo` field. Both changes
  are additive — the existing mock gateway's signature and every existing
  call site still typecheck unchanged.
- `src/features/payments/mombongo/mombongoProducerPaymentGateway.ts` —
  new gateway, same self-contained/idempotent-by-key structure as
  `mock/mockProducerPaymentGateway.ts`, returning the same response
  *fields* Mombongo's published contract documents (`providerRef`,
  `clientSecret` for card, `depositStatus: "ACCEPTED"` for mobile_money,
  `testMode: true`) — but resolved locally, no network call. Reuses the
  existing `mockScenarioController.ts` for deterministic
  pending/failure/timeout outcomes in tests, rather than duplicating it.
- `src/features/payments/getPaymentGateway.ts` — new resolution branch,
  gated by a second, independent `__DEV__`-only flag
  (`EXPO_PUBLIC_ENABLE_MOMBONGO_SIMULATION`). Demo wins if both flags are
  somehow on — documented, tested precedence.
- `src/features/guided/components/TestModeBanner.tsx` — new, distinct
  from `DemoDataBanner` (MOB-06B's own concept/copy, left untouched).
- `app/producer-invoice/[id]/pay.tsx` — the step machine branches on
  `capabilityMode`. `"demo"` is byte-for-byte unchanged (every existing
  MOB-06B test still passes against it, unmodified). `"real"` gets a new
  sequence: `verify → method → form → waiting → result` — method choice
  (card/mobile money; bank transfer shown disabled, matching Mombongo's
  own `501`), phone+operator collection for mobile money, a plain
  test-mode card-entry placeholder for card (no Stripe SDK added yet —
  there are no real Stripe keys to embed against), a non-dismissible
  waiting step (same hardware-back/`confirmNavigation` swallowing as the
  existing "processing" step), and real-shaped result copy (no
  "simulation" wording) paired with `TestModeBanner` so it's never
  mistaken for a connected payment.
- `app/producer-invoice/[id]/index.tsx` — the "Payer la facture" CTA now
  appears for both `mode: "demo"` and `mode: "real"` (previously
  `demo`-only), with `TestModeBanner` in place of `DemoDataBanner` for
  the real-simulation case.

## Test results (2026-08-31)

- **85 suites / 724 tests, all passing** (`bun run test`), including
  every existing MOB-06B demo-mode test unchanged, plus:
  - `mombongoProducerPaymentGateway.test.ts` (9 tests) — capability,
    scenario overrides, checkout-shape-per-method, idempotency, and a
    static import-hygiene check mirroring the mock gateway's own.
  - `PayInvoiceScreen.test.tsx` — 7 new tests for the `"real"` branch
    (method offering, mobile-money form validation, card's immediate
    enablement, waiting-step non-dismissibility + args passed to
    `startPayment`, result copy).
  - `getPaymentGateway.test.ts` — 4 new tests for the new branch's
    production lockout and demo-wins precedence.
- `bun run typecheck` and `bun run lint` both clean.
- One real bug caught and fixed during this pass: two new tests used
  `fireEvent.changeText(...)` without `await`, leaving a pending `act()`
  scope that bled into the next test ("overlapping act() calls" —
  intermittent failure only when run in sequence, not in isolation).
  Fixed by awaiting every `fireEvent` call, matching the rest of the file's
  convention.

## Visual verification

Same limitation as every prior sprint in this environment: no simulator/
Expo Go available. Conformance verified at code/test level only — a
manual Expo Go pass through the new method/form/waiting screens is still
recommended, particularly the mobile-money operator-chip layout and the
card placeholder screen's copy, before calling this simulation phase
visually done.

## Remaining for the real (non-simulated) pass

Everything in `MOB-07`'s revised scope: AROM-Production server routes
holding the two Mombongo secrets, the inbound webhook receiver, the
Firestore-listener-driven waiting step (currently a local timer, same
mechanism as the mock gateway), and — if/when card payment goes real —
`@stripe/stripe-react-native` for the actual Payment Sheet embed.
