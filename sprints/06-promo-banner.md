# Sprint 06 — Promo banner

**Status:** Todo

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

Proposed shape (confirm before building): a singleton doc
`config/promo` — mirrors the existing `config/parametres` singleton
pattern already in the data model —
`{active: boolean, headline: string, description: string, productId?: string, startDate?: string, endDate?: string}`.

## Contraintes

Same as sprint 05. `firestore.rules` needs a new `match /config/promo`
(or extend the existing `config/{doc}` match, which already covers this
path — verify before assuming a rules change is needed).

## Livrable

`AROM-Production` (dashboard promo editor, storefront banner component).
`AROM-Documentation` (data-model.md).

## Test de fumée

- [ ] Admin sets a promo active with a headline
- [ ] Banner appears at the top of `/storefront` for a partner (live, no refresh)
- [ ] Admin deactivates it — banner disappears
- [ ] Banner referencing a product from sprint 05 links/scrolls to it in the catalog
