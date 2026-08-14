# Sprint 14 — Storefront self-service: profile, order & product detail

**Status:** Done

**Rôle concerné :** Partenaire (utilise) / Admin (publie les fiches produit)
**Page / zone :** Storefront (`/storefront`, new `/storefront/profile`)

## Pourquoi maintenant

Requested directly, as three related gaps in the same page:

- No way for a partner to see or correct their own KYC/delivery info
  after onboarding (sprint 13) — no profile page existed at all.
- Clicking an order did nothing; a partner had no way to see the full
  breakdown, amount, or delivery status of a specific order beyond the
  one-line summary in the list.
- The catalogue was a bare list — no way to see more about a product
  than name/price, even though admin could already upload a photo
  (sprint 05). Not an online-store experience.

## Dans le périmètre

- **`/storefront/profile`**: a new route (not a modal, unlike checkout —
  this is a persistent, bookmarkable destination) showing the account
  email and assigned point of vente read-only, and an editable form for
  everything collected during onboarding (boutique name, contact name,
  phone, structured address, optional CNI/RCCM). Reuses
  `completePartnerOnboarding`'s field shape via a new
  `updatePartnerProfile` auth function.
- **Order detail sheet**: clicking an order in "Mes commandes" opens a
  bottom sheet with full line-item breakdown, status, the delivery
  estimate/date (see [sprints/07](<[DONE] 07-storefront-order-tracking.md>)), and
  the amount — phrased as "à payer à la livraison" for cash orders,
  "Payé" for completed mobile money.
- **Product detail sheet**: clicking a catalogue row (not the +/-
  stepper, which still adjusts quantity in place) opens a bottom sheet
  with a larger image, price, format, and the new `products.description`
  field (admin-editable, plain text) — this is what makes the catalogue
  read like an online store rather than a bare price list.
- `products.description` (optional) — admin types it in the same
  `CatalogueCard` row used for name/format/price/photo.

## Hors périmètre

- Product image galleries (multiple photos per product) — one photo,
  same as sprint 05.
- Structured product attributes (ingredients, nutritional info) beyond
  a single free-text description.
- Editing account **email** from the profile page — tied to Firebase
  Auth identity, not attempted here.

## Décisions déjà actées

- **Profile is a route, not a sheet.** Checkout and the two detail
  views are transient actions (modal-style bottom sheets); the profile
  is a persistent destination a partner might return to and bookmark —
  matches how `/storefront` itself is a route rather than a sheet.
- **`updatePartnerProfile` explicitly clears fields with `deleteField()`**
  rather than omitting the key — `completePartnerOnboarding` (sprint 13)
  only ever adds fields on first write, so omitting an empty optional
  field was fine there; here, a partner clearing a previously-set
  `idNumber` needs it to actually disappear, not silently keep its old
  value (an `updateDoc` with a key omitted leaves that field untouched).
- **Clicking a catalogue row opens the detail sheet; the qty stepper
  stays a separate, sibling control** — so adding to cart doesn't
  require opening and closing a sheet for the common case, matching how
  the checkout flow (sprint 08) already separated "browse" from "act."

## Contraintes

Same as sprint 05. No `firestore.rules` changes for the profile edit or
product description — both fall under rules that already permit these
writes (`users/{uid}` partner self-update, `products/{id}` admin/staff
write) — see [data-model.md](data-model.md#accounts--storefront).

## Livrable

`AROM-Production` (`storefront/profile.tsx`; `OrderDetailSheet.tsx`,
`ProductDetailSheet.tsx`; `auth.tsx` — `updatePartnerProfile`;
`dashboard.tsx` — `CatalogueCard` description field). `AROM-Documentation`
(data-model.md).

## Test de fumée

- [x] Profile page pre-fills every field collected during onboarding
- [x] Editing a field and saving persists it and shows a confirmation
- [x] Clicking an order opens a detail sheet with status, delivery
      estimate/date, and the correct amount for its payment method
- [x] Clicking a catalogue product opens a detail sheet with its photo,
      price, and admin-set description; the quantity stepper inside it
      stays in sync with the cart
- [x] A description set in the admin catalogue appears on the storefront
      detail sheet live

Covered by the automated regression suite (`node regression-suite.mjs`).
