# Sprint 07 — Storefront order tracking

**Status:** Todo

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

## Livrable

`AROM-Production` (`orders.deliveryDate` field, admin input on confirm
in `OrdersCard`, storefront order card display). `AROM-Documentation`
(data-model.md).

## Test de fumée

- [ ] Admin confirms an order and sets a delivery date
- [ ] Partner sees that date on their order card without refreshing
- [ ] Admin changes the date — partner sees the update live
- [ ] An order with no delivery date yet still displays sensibly (no "Invalid Date")
