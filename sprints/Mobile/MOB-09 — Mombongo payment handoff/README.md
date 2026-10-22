# MOB-09 — Real Mombongo payment gateway (revised against the real API contract)

> **Status: Done** (verified 2026-09-01, this update). The previous note
> on this line flagged the "waiting step driven by a Firestore listener,
> not polling" acceptance criterion as unmet — that gap is now closed.
> `producer-invoice/[id]/pay.tsx` subscribes to `onSnapshot` on the
> invoice doc for the whole real-mode wait (from `waiting` through a
> `pending` result), torn down once a final outcome is known or the admin
> navigates away — same lifecycle discipline as `lib/firebase/auth.tsx`'s
> own `onSnapshot` precedent. It only listens for `statut === "payee"`:
> `AROM-Production/src/routes/api/webhooks/mombongo.ts` only ever writes
> on `status === "paid"` (an unpaid/failed status is acknowledged and
> logged, never applied), so there is no live push signal for failure
> today — a stuck payment is still only surfaced by the existing 2h
> threshold, not by this listener. Also added since the prior update: a
> "Voir le reçu" receipt view (`producer-invoice/[id]/receipt.tsx`,
> reachable from the real success result) and a stuck-payment item in
> `adminPriorities.ts` (`invoice-stuck-{id}`, urgent, same 2h threshold
> as the detail screen's own banner — both now read from the shared
> `MOMBONGO_STUCK_THRESHOLD_HOURS` in `features/invoices/model.ts`
> instead of a locally duplicated constant). `bun run typecheck`, `lint`,
> `test` all pass (959 tests, including new coverage for the listener,
> the receipt screen, and the stuck-priority item).

## Revision note (2026-08-31)

Originally scoped as a redirect/WebView handoff to a Mombongo-hosted
checkout page. **That page doesn't exist and isn't the actual contract.**
Mombongo's real `createExternalInvoiceCheckout` response (their §3) is:

- `card` → a Stripe `clientSecret` — meant to be **embedded directly in
  AROM's own UI** via a Stripe SDK (`@stripe/stripe-react-native`), not a
  redirect.
- `mobile_money` → requires `phone` + `operator` (`mpesa`/`airtel`/
  `orange`) **in the request itself**, and returns `depositStatus:
  "ACCEPTED"` — an on-device USSD/PIN prompt the payer completes on
  their own phone, not a URL AROM opens.
- `bank_transfer` → `501`, not implemented on Mombongo's side yet.

So this sprint is "collect payment details in AROM's own wizard, submit
them, then wait for a real webhook" — closer to how the old PawaPay stub
felt to the user (a form in AROM's own UI) than to a handoff. The
reliability win over that old stub still holds: **wait via a Firestore
listener on `producerInvoices.statut`/`mombongoCheckout`**, driven by
MOB-07's webhook receiver, not client-side polling.

## Objective

Replace `MockProducerPaymentGateway` with a real
`MombongoProducerPaymentGateway` implementing the exact same
`ProducerPaymentGateway` interface MOB-06B already built
(`src/features/payments/gateway.ts` — `getCapability()`/`startPayment()`,
`PaymentGatewayMode` already reserves `"real"`), and update the existing
wizard (`app/producer-invoice/[id]/pay.tsx`) to collect what the real
contract actually needs.

## Technical approach

1. **`src/features/payments/mombongo/mombongoProducerPaymentGateway.ts`**
   — real implementation:
   - `getCapability()`: `{ available: true, mode: "real" }` once MOB-07's
     server routes are reachable and provisioned; `unavailable` otherwise
     — same shape as the existing mock, different resolution condition
     (server-side capability, not `__DEV__` + an env flag).
   - `startPayment({ invoiceId, idempotencyKey })`: calls MOB-07's
     `createMombongoCheckout` server function with whatever method the
     wizard already collected (see below), returns a `PaymentResult` — but
     unlike the mock, `"success"` here should really mean "the checkout
     was accepted by Mombongo," **not** "the payment is confirmed" — the
     wizard's waiting step (not this function) is what actually learns
     the final outcome, from the Firestore listener.
2. **`getPaymentGateway.ts`**: extend the seam — real mode resolves when
   MOB-07's infra is provisioned in the current environment (a new check,
   distinct from the existing `__DEV__ && EXPO_PUBLIC_ENABLE_MOCK_PAYMENTS`
   dev-only gate, which stays exactly as-is for whoever still wants the
   pure simulation). Both can coexist: dev/demo mode still exists for
   internal testing without touching Mombongo at all; real mode is the
   new, separate branch.
3. **Wizard changes** (`app/producer-invoice/[id]/pay.tsx`), step machine
   extended:
   - **Method choice**: `card` and `mobile_money` only (`bank_transfer`
     shown but disabled with "Bientôt disponible" — matches Mombongo's
     own `501`, don't build UI for a rail that doesn't exist yet).
   - **`mobile_money` chosen**: collect `phone` + `operator` in-app
     (reuse form patterns already established elsewhere, e.g.
     `ProducerStep.tsx`'s phone input) before calling `startPayment`.
   - **`card` chosen**: embed Stripe's Payment Sheet
     (`@stripe/stripe-react-native` — new dependency, confirm it's
     Expo-Go-compatible or requires a dev client build, since some native
     Stripe SDK features don't run in Expo Go) using the returned
     `clientSecret`.
   - **Waiting step**: exactly as before — non-dismissible, hardware-back
     swallowed, `confirmNavigation` — but now listens to the real
     `producerInvoices` doc via Firestore's `onSnapshot`, not a mock
     timer.
   - **Result step**: real `statut: "payee"` → success; still `pending`
     past a reasonable window → pending state with a safe "check back
     later" action; explicit `failed` (if Mombongo's webhook ever reports
     that shape — confirm with them whether `status` can be anything
     other than `paid`) → failure state.
4. **Duplicate-submit guard**: enforced server-side in MOB-07 (a second
   checkout call for an invoice already `checkout_created` gets Mombongo's
   own `409`) — the wizard just needs to surface that cleanly, not
   re-implement the guard client-side.
5. **`testMode`**: every checkout created while AROM's own test flag is on
   should carry Mombongo's `testMode: true` — verify the wizard can't
   accidentally create a real (non-test) checkout while AROM's own gate
   is still closed.

## Acceptance criteria

- Card payment: Stripe Payment Sheet opens with a real `clientSecret`,
  completes, and the wizard's waiting step correctly reflects the
  Firestore-driven outcome once MOB-07's webhook fires.
- Mobile money: phone+operator collected in-app, `startPayment` returns
  `ACCEPTED`-equivalent, waiting step behaves identically to the card
  path from that point on (same listener, same result screen).
- Bank transfer is visibly disabled, not hidden — matches the shared
  guided-screen contract's "explain why, don't just remove" rule already
  established in MOB-06A/B for unsupported states.
- A second payment attempt on an in-flight invoice is blocked with a
  clear message, not a silent duplicate.
- The waiting step survives the app being backgrounded and resumed
  (explicit test — this is the exact PawaPay-stub failure mode being
  fixed).
- `bun run typecheck`, `bun run lint`, `bun run test` pass, including a
  new gateway implementation test suite mirroring
  `mockProducerPaymentGateway.test.ts`'s structure.

## Out of scope

- Bank transfer (Mombongo's `501` is authoritative).
- FC→USD conversion (MOB-07).
- The result-state copy polish / broader reconciliation question — see
  MOB-10 (now mostly resolved, not open, per its own revision).

## Open question carried over from Mombongo's contract

Their webhook payload's `status` field is documented as `"paid"` in
every example shown — confirm with them whether `"failed"` (or any other
value) is a real possibility before building a failure branch that may
never actually fire.
