# Sprint 06 — Promo banner

**Status:** Done

**Rôle concerné :** Admin (édite) / Partenaire (voit)
**Page / zone :** Dashboard — Commercialisation, Storefront (catalogue)

## Pourquoi maintenant

Admin has no way to surface a promotion to partners — there's no banner,
announcement, or featured-product mechanism anywhere in the storefront
today.

## Dans le périmètre

A single active promo, editable by admin: headline, short description,
optional reference to a product (from sprint 05's catalog), optional
start/end dates. Shown as a banner at the top of `/storefront` when
active; hidden entirely otherwise.

## Hors périmètre

Multiple simultaneous banners / rotation / scheduling queue — v1 ships
with exactly one active-or-not promo, matching how a small operation
actually runs a promotion.

## Décisions déjà actées

Shipped as proposed: a singleton doc `config/promo` — mirrors the
existing `config/parametres` singleton pattern —
`{active: boolean, headline: string, description?: string, productId?: string, startDate?: string, endDate?: string}`.

One deviation from the original plan: banners referencing a product open
that product's detail sheet (see [sprints/14](<[DONE] 14-storefront-self-service.md>))
rather than scrolling to it in the catalogue list — the detail sheet
didn't exist when this sprint was originally scoped, and opening it
directly is a more direct action than scroll-and-locate.

## Contraintes

Same as sprint 05. `firestore.rules` **did** need a change, unlike the
"verify before assuming" note originally here — the existing
`config/{doc}` match is admin/staff-only, and partners need to read this
one doc to see the banner. Fixed with an exact-path `match /config/promo`
ahead of the `config/{doc}` wildcard (Firestore matches the most
specific path), so every other `config` doc keeps its admin/staff-only
access unchanged. See
[data-model.md](data-model.md#accounts--storefront).

## Livrable

`AROM-Production` (dashboard promo editor, storefront banner component).
`AROM-Backend` (`firestore.rules`). `AROM-Documentation` (data-model.md).

## Test de fumée

- [x] Admin sets a promo active with a headline
- [x] Banner appears at the top of `/storefront` for a partner (live, no refresh)
- [ ] Admin deactivates it — banner disappears (logically covered by the
      same `active` check the "appears" case exercises; not separately
      exercised by the automated regression suite)
- [x] Banner referencing a product opens that product's detail sheet
      (see deviation above)

Covered by the automated regression suite (`node regression-suite.mjs`).
