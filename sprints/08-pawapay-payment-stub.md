# Sprint 08 — PawaPay mobile money (stub)

**Status:** Todo

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

- A `/api/pawapay/initiate` server route (Nitro, same pattern as the
  existing `[.mcp]` server routes) holding the API token server-side.
- A status-check proxy route the browser polls.
- At checkout, the partner chooses "Payer par mobile money" or "Payer à
  la livraison" (today's unchanged manual flow).
- `orders.payment` sub-object tracks method/status/depositId/provider/amount.
- On `COMPLETED`, the browser writes the payment status to its own order
  (client-side polling — see decision below) and the order→`ventes`
  bridge (sprint 01) uses the real paid amount for `encaisse` instead of
  the hardcoded `0`.

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
browser — this is the reason the server route exists at all, not an
optional hardening step.

## Livrable

`AROM-Production` (server routes, storefront checkout UI, dashboard
payment status display). `AROM-Documentation` (data-model.md: `orders.payment`
shape; flows.md updated once real sandbox credentials are in and the
stub is swapped out).

## Test de fumée

- [ ] Partner places an order, chooses "Payer par mobile money"
- [ ] Stub returns `ACCEPTED`, then (simulated) `COMPLETED`
- [ ] Order shows "Payé" on the partner's storefront view
- [ ] Admin fulfills — the resulting `ventes` row has the real paid
      amount in `encaisse`, not `0`
- [ ] Choosing "Payer à la livraison" still works exactly as it does today
