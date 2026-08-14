# Sprint 01 — Order → ventes bridge

**Status:** Done — [Teddmab/AROM-Production#1](https://github.com/Teddmab/AROM-Production/pull/1)

**Rôle concerné :** Admin / Staff
**Page / zone :** Dashboard — Commercialisation

## Pourquoi maintenant

Confirming/fulfilling a storefront order didn't create a corresponding
`ventes` row, so storefront sales never showed up in the dashboard's
commercial KPIs — flagged in the roadmap as the highest-value gap once
the storefront saw any real traffic.

## Dans le périmètre

An order-management panel in the dashboard's Commercialisation section:
staff can confirm/cancel pending orders, and marking one "Livrée"
atomically closes the order and writes one `ventes` row per line item.

## Hors périmètre

Payment (see sprint 08), delivery scheduling (see sprint 07), product
images (see sprint 05).

## Décisions déjà actées

- `canal` on the generated `ventes` row is always `"Grossiste"` —
  distinguishes storefront/partner sales from manually-entered channels.
- `encaisse` is `0` on the generated row — storefront orders aren't a
  checkout yet, so payment status is reconciled manually like any other
  credit sale (this is what sprint 08 changes).
- Vente doc IDs are deterministic (`VTE-ORD-<orderId>-<itemIndex>`), so
  re-running the conversion overwrites instead of duplicating.

## Contraintes

bun, not npm, in `AROM-Production`. `firestore.rules` is the source of
truth for access control. Test against the Firebase emulator before
touching live data. PR against `main`, no direct push.

## Livrable

`AROM-Production` (dashboard `OrdersCard`, order item now snapshots
product `format`), `AROM-Backend` (emulator support in admin scripts, to
make the above testable locally), `AROM-Documentation`
(data-model.md, roadmap.md).

## Test de fumée

- [x] Partner places a multi-line order
- [x] Admin confirms, then marks it "Livrée"
- [x] Two `ventes` rows appear with correct client/canal/format/quantity/price
- [x] `bouteillesVendues`/`ca`/Grossiste-channel KPIs update accordingly
- [x] Re-running the conversion doesn't create duplicate `ventes` rows
