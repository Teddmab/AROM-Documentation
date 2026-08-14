# v1 — MVP: the complete commerce loop

## What "done" means for v1

Every role can do their whole job through the app, with no dead ends and
no hand-editing Firestore:

```
Admin publishes a product (name, price, photo) and, optionally, a promo
        │
        ▼
Partner sees the live catalog + banner on /storefront
        │
        ▼
Partner orders, and pays either by PawaPay mobile money or on delivery
        │
        ▼
Admin confirms, sets a delivery date, fulfills
        │
        ▼
A `ventes` row lands with the real amount paid — commercial KPIs are
correct without anyone touching the Firestore console
```

This is the same diagram as [flows.md](../flows.md#target-flow); v1 is
the milestone where it stops being a diagram and starts being true.

**Explicitly out of v1** — real gaps, deliberately not blocking this
milestone (see [roadmap.md](../roadmap.md) for the full backlog):

- Staff per-collection data scoping (`menus` is UI-only today).
- Fixed/recurring delivery schedule (v1 ships with an ad-hoc delivery
  date the admin sets by hand, per the decision in sprint 07).
- PawaPay server-side webhook (v1 ships with client-side polling —
  correct on purpose, see sprint 08's "Décisions déjà actées").

Admin/staff invite-link signup ([sprint 09](<[DONE] 09-invite-link-admin-staff-signup.md>))
shipped ahead of v1, outside the commerce loop above — it turned out not
to need the Cloud Functions decision it was originally scoped against.

## Sprints

| # | Sprint | Role | Page | Status |
| --- | --- | --- | --- | --- |
| 01 | [Order → ventes bridge](<[DONE] 01-order-ventes-bridge.md>) | Admin/Staff | Dashboard — Commercialisation | Done |
| 02 | [iOS-style redesign & branding](<[DONE] 02-ios-redesign-branding.md>) | Tous | Landing, Login, Signup, Storefront | Done |
| 03 | [Bug fixes, responsive audit, landing animations](<[DONE] 03-bugfixes-responsive-animations.md>) | Visiteur/Partenaire | Landing, Login, Signup | Done |
| 04 | [Account-lockout & signup session fix](<[DONE] 04-account-lockout-fix.md>) | Tous | Login, toutes les pages protégées | Done |
| 05 | [Admin catalog management](<[DONE] 05-admin-catalog-management.md>) | Admin/Staff | Dashboard — Commercialisation | Done |
| 06 | [Promo banner](<[DONE] 06-promo-banner.md>) | Admin → Partenaire | Dashboard, Storefront | Done |
| 07 | [Storefront order tracking](<[DONE] 07-storefront-order-tracking.md>) | Partenaire/Admin | Storefront, Dashboard | Done |
| 08 | [PawaPay mobile money (stub)](<[DONE] 08-pawapay-payment-stub.md>) | Partenaire/Admin | Storefront checkout, Dashboard | Done — stub phase, real PawaPay credentials still pending |

**v1's core commerce loop is complete as of 2026-08-14** — every sprint
in the table above is Done. "PawaPay (sandbox)" in the smoke test below
means the stub, which mimics the real API's request/response shape; the
loop works end to end, just not yet against real PawaPay credentials
(that swap is scoped as its own follow-up, not a v1 blocker).

## v1 end-to-end smoke test

Beyond each sprint's own smoke test, v1 as a whole isn't done until this
single walkthrough works without touching the Firestore console or the
CLI scripts:

- [x] As **admin**, publish a new product with a photo and a price
- [x] Set a promo banner mentioning that product
- [x] As a **partner** (fresh signup), see the catalog with the photo and
      the banner on `/storefront`
- [x] Place an order and pay via PawaPay (sandbox)
- [x] As **admin**, confirm the order, set a delivery date, then mark it
      fulfilled
- [x] Confirm the `ventes` row shows the real amount paid and the
      Commercialisation KPIs reflect it
- [x] As the **partner**, see the order's final status and delivery date
      without refreshing

Covered by the automated regression suite (`node regression-suite.mjs`,
21/21 passing as of this update).
