# Sprint 07 — Storefront order tracking

**Status:** Done

**Rôle concerné :** Partenaire / Admin
**Page / zone :** Storefront (mes commandes), Dashboard — Commercialisation

## Pourquoi maintenant

A partner can currently only see `pending`/`confirmed`/`fulfilled`/
`cancelled` on their own order — no sense of whether it's been accepted
for real, or when to expect delivery.

## Dans le périmètre

When admin confirms an order, they set a delivery date by hand (no fixed
recurring schedule — this was decided explicitly, see below). The
partner's order card on `/storefront` shows that date alongside the
status, live.

## Hors périmètre

A fixed/recurring delivery cadence (e.g. "every Friday") — not modeled.
Partial fulfillment / split deliveries. Push/SMS notifications on status
change (Firestore `onSnapshot` already makes the in-app update live,
which covers the case of the partner having the page open).

## Décisions déjà actées

Ad-hoc delivery date, set manually by admin — decided explicitly earlier
in this project rather than building a scheduling system nobody asked
for yet. Revisit only if AROM settles into an actual fixed delivery
cadence operationally.

## Contraintes

Same as sprint 05.

Also picked up along the way: `orders.fulfilledAt` is now set when an
order is marked "livrée" (previously nothing recorded when that
happened), so the storefront order detail sheet (see
[sprints/14](<[DONE] 14-storefront-self-service.md>)) can show a real "Livrée le
…" date instead of just the status label.

## Livrable

`AROM-Production` (`orders.deliveryDate`/`fulfilledAt` fields, admin
date input on confirm in `OrdersCard`, storefront order card + detail
sheet display). `AROM-Documentation` (data-model.md).

## Test de fumée

- [x] Admin confirms an order and sets a delivery date
- [x] Partner sees that date on their order card without refreshing
- [ ] Admin changes the date after confirming — **not built**: the date
      input is only shown at confirm time, matching the "ad-hoc, set
      manually" decision above but with no edit-after-confirm path yet.
      If this turns out to matter operationally, it's a small addition
      (same inline-input pattern, shown for `confirmed` rows too).
- [x] An order with no delivery date yet still displays sensibly — the
      date line is conditionally rendered (`o.deliveryDate &&`) rather
      than ever calling `new Date(undefined)`, and date-only strings are
      formatted without `Date` timezone conversion (`formatDateOnly` in
      `AROM-Production/src/lib/erp/model.ts`) to avoid an off-by-one-day
      display bug across timezones

Covered by the automated regression suite (`node regression-suite.mjs`).
