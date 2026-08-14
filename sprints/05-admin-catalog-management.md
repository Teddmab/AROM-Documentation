# Sprint 05 — Admin catalog management

**Status:** Todo

**Rôle concerné :** Admin / Staff
**Page / zone :** Dashboard — Commercialisation (nouvelle carte "Catalogue")

## Pourquoi maintenant

`products` is only ever written by `AROM-Backend/scripts/seed.mjs` or by
hand in the Firestore console — there's no way to add, price,
deactivate, or photograph a product from the app itself, even though
`storage.rules` already reserves `products/**` for exactly this
(confirmed: `allow read: if true; allow write: if ... role() in
['admin', 'staff']` — no rule changes needed). This is the first,
largest gap in the target flow (see [flows.md](../flows.md)).

## Dans le périmètre

A "Catalogue" card in the dashboard's Commercialisation section: list
existing products, create a new one (name, format, price, active
toggle), edit an existing one, upload/replace a photo to Firebase
Storage under `products/{id}`. Changes are live on `/storefront`
immediately (it already reads `products` via `onSnapshot`).

## Hors périmètre

Promo banner (sprint 06). Multiple photos per product (one image field
for v1). Product categories/variants beyond the existing three bottle
formats.

## Décisions déjà actées

- `products/{id}` gets a new `imageUrl` field (nullable — existing
  seeded products have none until an admin uploads one).
- Storage path convention: `products/{productId}/photo` (single file,
  overwritten on replace, matching `storage.rules`' existing
  `products/{allPaths=**}` match).

## Contraintes

bun, not npm. Test against the Firebase emulator (Storage emulator too —
not yet used by any local dev workflow, needs adding to the
`firebase emulators:start` command in the runbook). PR against `main`.
No `firestore.rules`/`storage.rules` changes needed — verify this stays
true rather than assuming it.

## Livrable

`AROM-Production` (dashboard Catalogue card, storefront product card
image display). `AROM-Documentation` (data-model.md: `imageUrl` field;
runbook.md: Storage emulator in the local dev flow if it wasn't already
covered).

## Test de fumée

- [ ] As admin, create a new product with a name, price, and photo
- [ ] It appears on `/storefront` with the photo, immediately, no refresh
- [ ] Edit its price — the storefront reflects the new price live
- [ ] Deactivate it — it disappears from the storefront catalog
- [ ] Existing partner orders referencing the old product are unaffected
      (orders snapshot product data at order time, per sprint 01)
