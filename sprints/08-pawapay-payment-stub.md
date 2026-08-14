# Sprint 08 — PawaPay mobile money (stub)

**Status:** Done

**Rôle concerné :** Partenaire (paie) / Admin (rapproche)
**Page / zone :** Storefront checkout, Dashboard — Commercialisation

## Pourquoi maintenant

Orders currently go straight from pending → confirmed → fulfilled with
no payment concept — the order→`ventes` bridge (sprint 01) always writes
`encaisse: 0`. AROM operates in the Kasaï (DRC), where mobile money is
the practical payment rail.

## Dans le périmètre

The architecture from [flows.md](../flows.md#payment--pawapay-mobile-money),
built against a stub (no real PawaPay credentials yet):

- Two TanStack Start server functions (`createServerFn`, in
  `AROM-Production/src/lib/payments/pawapay.ts`) hold the API token
  server-side — not a Nitro `/api` route as originally sketched, since
  the `[.mcp]` server-route pattern this sprint was meant to follow was
  removed in the Lovable-dependency cleanup (sprint 11). `createServerFn`
  gives the same guarantee (handler code never ships to the browser)
  with less boilerplate.
- The storefront (`AROM-Production/src/routes/storefront/index.tsx`) is
  reorganized into two tabs — Catalogue and Mes commandes — and
  "Commander" opens a checkout sheet
  (`AROM-Production/src/components/storefront/CheckoutSheet.tsx`)
  instead of placing the order directly.
- At checkout, the partner chooses "Mobile money" or "Paiement à la
  livraison" (today's unchanged manual flow, renamed for clarity).
- `orders.payment` sub-object tracks method/status/depositId/provider/amount
  — see [data-model.md](../data-model.md#payment-sprint-08-stub-phase).
- The order document is only created after the outcome is known: cash on
  delivery creates immediately (`status: "pending"`), mobile money polls
  until `COMPLETED` before creating the order at all. This means no
  `firestore.rules` changes were needed — see decisions below.
- The order→`ventes` bridge (sprint 01) now uses the real paid amount
  for `encaisse` instead of the hardcoded `0`.

## Hors périmètre

Server-side webhook (needs a Firebase Admin SDK service account in the
Cloudflare Worker — deliberately deferred, same fork as admin invites,
sprint TBD). Real sandbox/production PawaPay credentials — this sprint
ships against a local stub matching PawaPay's real request/response
shapes; swapping in real credentials should need no rework.

## Décisions déjà actées

- **Coexistence, not replacement** — mobile money is one of two payment
  paths, not a requirement to place an order.
- **Client-side polling over server webhook** for this phase — no new
  server credentials needed, matches "no Cloud Functions yet." Traded
  off against reliability if the tab closes right after paying; accepted
  for a stub/testing phase.
- DRC providers to support: `VODACOM_MPESA_COD`, `AIRTEL_COD`,
  `ORANGE_COD`; currency `CDF` (matches AROM's "FC" pricing).
- The exact webhook/callback payload shape was **not** confirmed during
  research — must be verified against a real sandbox account before any
  server-side webhook work (out of scope here) is built on top of it.

## Contraintes

Same as sprint 05. The PawaPay Bearer token must never reach the
browser — this is the reason the server functions exist at all, not an
optional hardening step.

## Livrable

`AROM-Production` (server functions, storefront tabs + checkout UI,
dashboard payment status display). `AROM-Documentation` (data-model.md:
`orders.payment` shape; flows.md updated once real sandbox credentials
are in and the stub is swapped out).

## Test de fumée

- [x] Partner places an order, chooses "Mobile money"
- [x] Stub returns `ACCEPTED`, then (via client-side polling) `COMPLETED`
- [x] Order shows "Payé par mobile money" on the partner's storefront
      "Mes commandes" tab
- [x] Admin fulfills — the resulting `ventes` row has the real paid
      amount in `encaisse`, not `0`
- [x] Choosing "Paiement à la livraison" still works, order created
      immediately with `status: "pending"`

Covered by the automated regression suite (`node regression-suite.mjs`),
extended in this sprint with cash-on-delivery and mobile-money checkout
cases — 12/12 passing.
