# Firestore data model

Project: `arom-production`. Native mode, `eur3`. All collections are
top-level (no subcollections) — document IDs are the same client-generated
IDs the original `localStorage` model used (`newId(prefix)` in
`AROM-Production/src/lib/erp/store.tsx`), except `products` (fixed catalog
IDs) and `orders`/`users` (Firestore/Auth-generated IDs).

## ERP campaign data

Shape mirrors `AROM-Production/src/lib/erp/model.ts`'s `ErpState` — one
collection per array field, plus a singleton config doc:

| Collection / doc | Fields | Notes |
| --- | --- | --- |
| `config/parametres` | `objectifAnanasKg`, `objectifBouteilles`, `prix500/330/300`, `tauxCommission`, `tauxPrimeProduction`, `tauxPertesMax`, `distanceFournisseurKm`, `debutProduction`, `finProduction`, `finCommercialisation`, `objectifClients`, `objectifMargeBrute` | Single doc, edited from the "Paramètres ERP" dashboard section |
| `producteurs/{id}` | `nom`, `village`, `secteur`, `territoire`, `telephone`, `produit`, `capaciteKgMois`, `prixConvenu`, `statut`, `observations` | Supplier registry |
| `approvisionnements/{id}` | `numero`, `date`, `idProducteur`, `fournisseur`, `village`, `produit`, `qteCommandeeKg`, `qteRecueKg`, `prixKg`, `transport`, `autresFrais`, `qualite` | Purchase receipts |
| `productions/{id}` | `lot`, `date`, `kgUtilises`, `volumeJusL`, `q500`, `q330`, `q300`, `rejets`, `responsable`, `staffUid?`, `statut` | Production batches. `responsable` and `staffUid` (sprint 17) are auto-filled from the logged-in staff account, not typed — `staffUid` powers per-person bonus tracking; entries predating sprint 17 have no `staffUid` |
| `stockMP/{id}` | `date`, `produit`, `unite`, `type` (`Entrée`/`Sortie`/`Ajustement`), `entree`, `sortie`, `coutUnitaire`, `observation` | Raw-material stock ledger |
| `clients/{id}` | `nom`, `categorie`, `contact`, `zone`, `premierContact`, `statut` | Internal client registry (distinct from `users`/partner accounts) |
| `ventes/{id}` | `numero`, `date`, `idClient`, `client`, `canal`, `format`, `quantite`, `prixUnitaire`, `remise`, `encaisse`, `commerciale`, `staffUid?` | Sales journal. `commerciale`/`staffUid` (sprint 17) — same auto-fill as `productions.responsable`/`staffUid` above |
| `marketing/{id}` | `numero`, `date`, `campagne`, `canal`, `cible`, `description`, `budget`, `coutReel`, `contacts`, `prospects`, `ventesGenerees` | Marketing actions |
| `charges/{id}` | `rubrique`, `budget`, `realise` | Fixed charges |

All derived KPIs (marge brute, rendement, primes, etc.) are computed
client-side from these by `AROM-Production/src/lib/erp/engine.ts` — none
of that is stored.

## Accounts & storefront

| Collection | Fields | Notes |
| --- | --- | --- |
| `users/{uid}` | `email`, `displayName`, `role` (`admin`\|`staff`\|`partner`), `menus` (`"all"` or `string[]`), `active`, `createdAt`, `inviteId?`, `onboardingComplete?`, `contactName?`, `phone?`, `address? {ville, commune, quartier, repere?}`, `idNumber?`, `pointDeVente?`, `verified?`, `poste?` | `uid` matches the Firebase Auth UID. `inviteId` is only present on admin/staff accounts created via `/join` (see `invites` below); an audit trail, not read by any rule. The `onboardingComplete` through `verified` fields are partner-only. `onboardingComplete` through `pointDeVente` are written by the guided onboarding wizard — see below. `verified` (sprint 16) defaults `false` at signup and is flipped by admin from the dashboard's "Boutiques partenaires" card after a phone-call confirmation — informational only, never a gate on ordering. `idNumber` was write-only (collected at signup, never read back by the dashboard) until [sprint 18](<sprints/[DONE] 18-dashboard-ia-data-rethink.md>), which surfaces it next to the "Vérifié" toggle so admin has something to actually review before verifying. `poste` (sprint 17, staff-only) is `"Directeur de Production"` \| `"Chargée de Commercialisation"` \| `"Personnalisé"` — see below |
| `invites/{id}` | `email`, `role` (`admin`\|`staff`), `menus` (`"all"` or `string[]`), `poste?`, `used`, `usedBy?`, `createdBy`, `createdAt` | Lets an admin grant admin/staff access without the CLI — see [rbac.md](rbac.md#how-adminstaff-accounts-are-provisioned). Single-use, email-locked; doc ID is the actual secret (get-by-ID is public, listing is admin-only). `poste` (sprint 17) carries through to the redeemed account's `users/{uid}` doc, validated the same way as `role`/`menus` — see below |
| `products/{id}` | `name`, `format`, `price`, `active`, `imageUrl?`, `description?` | Storefront catalog; seeded with the three bottle formats from `parametres`. `imageUrl` is optional (nullable) — populated by uploading a photo in the dashboard's Catalogue card, which stores it in Storage at `products/{id}/photo` and writes the resulting download URL here. Fully admin/staff-manageable from the dashboard as of [sprints/05](<sprints/[DONE] 05-admin-catalog-management.md>) — not just seed-script-only anymore. `description` (sprint 14) is free text shown on the storefront's product detail sheet — no structured attributes (ingredients, nutrition) yet |
| `orders/{id}` | `partnerId`, `partnerName`, `partnerPhone?`, `partnerAddress?`, `items: {productId, name, quantity, unitPrice, format}[]`, `total`, `status` (`pending`\|`confirmed`\|`fulfilled`\|`cancelled`), `createdAt`, `deliveryDate?`, `fulfilledAt?`, `payment: {method, status, ...}` | One doc per storefront order. `format` is a snapshot of the product's format at order time (not a live join), matching how `name`/`unitPrice` are already snapshotted. `partnerPhone`/`partnerAddress` are likewise a snapshot of the partner's profile at order time (a single formatted string for the address, not the structured object) — see below. `deliveryDate` (sprint 07) is set by hand by admin when confirming an order — ad-hoc, not a recurring schedule. `fulfilledAt` (sprint 07) is stamped automatically when an order is marked "livrée," so the storefront can show a real date instead of just the status. `payment` is written once at order creation and never mutated afterward — see below |
| `config/promo` | `active: boolean`, `headline: string`, `description?: string`, `productId?: string`, `startDate?: string`, `endDate?: string` | Single active-or-not storefront promo (sprint 06) — no rotation/multiple-banner queue. Shown on `/storefront` when `active` and (if set) within `startDate`/`endDate`. Admin-only write, but **signed-in read** (unlike `config/parametres`, which is admin/staff-only) — see [rbac.md](rbac.md) and the `config/promo` exact-path rule ahead of the `config/{doc}` wildcard in `firestore.rules` |

`firestore.indexes.json` defines a composite index on
`orders(partnerId ASC, createdAt DESC)` for the "my orders" query.

## Guided boutique onboarding (sprint 13)

`/storefront/signup` (`AROM-Production/src/routes/storefront/signup.tsx`)
is a 5-step wizard rather than a single form, so a non-technical partner
is walked through it one question at a time:

1. **Compte** — business name, email/password (or Google/Facebook OAuth).
2. **Contact** — `contactName` (the person to call), `phone`.
3. **Localisation** — structured `address`: `ville` (dropdown of known
   Kasaï-region towns + "Autre" free text), `commune`, `quartier`, and an
   optional `repere` (landmark) — DRC addresses commonly lack formal
   street addressing, so this is deliberately not a single free-text field.
4. **Identification** — optional `idNumber` (CNI or RCCM), light-touch
   KYC. Explicitly not a document-upload + admin-review flow (out of
   scope for this sprint) — no new Storage rules were needed.
5. **Résumé** — review screen, shows the depot the boutique will be
   served from, then writes everything in one `completePartnerOnboarding`
   call.

**Two-write account creation.** Step 1 creates the Firebase Auth account
and an immediate `users/{uid}` doc with `onboardingComplete: false` (via
`signUpPartner`/`signUpPartnerWithProvider` — unchanged from before this
sprint except for that one field). Steps 2–4 are held in local component
state, not written per-step. Step 5's `completePartnerOnboarding` does
one `updateDoc` with everything collected plus `onboardingComplete: true`
and `pointDeVente`. This two-write shape (not five) exists specifically
to avoid the alternative of a signed-in Auth account with **no**
`users/{uid}` doc at all if someone abandons the wizard early —
`RequireRole` treats that state as "Accès indisponible", a dead end that
signs the user out. A doc always exists from the end of step 1 onward,
so an abandoned wizard is resumable: returning to `/storefront/signup`
(or reaching `/storefront` directly, which redirects back) picks up at
step 2 with `displayName`/`email` already known.

**No `firestore.rules` change needed.** The existing partner
self-`update` rule only constrains `role` from changing — it doesn't
enumerate field keys — so writing these new fields to a partner's own
doc needed no rules change, matching how sprint 08's `orders.payment`
needed none either.

**Point of sale (stub).** AROM currently delivers from a single point of
sale. Rather than a `pointsDeVente` collection with commune→depot
matching logic for exactly one entry, `pointDeVente` is a constant
(`AROM_DEPOT_NAME` in `AROM-Production/src/lib/storefront/depot.ts`)
written onto the partner's profile at onboarding. Replace this with a
real collection + lookup when AROM adds a second delivery point.

## Payment (sprint 08, stub phase)

`orders.payment` is one of:

- `{ method: "cash_on_delivery", status: "pending" }` — set at order
  creation, unchanged until the order is marked "livrée" (see below).
- `{ method: "pawapay", status: "completed", depositId, provider, amount, currency: "CDF", updatedAt }`
  — the order document is **only ever written after** the PawaPay
  deposit reaches `COMPLETED`; there is no `orders` doc for a deposit
  that's still pending or that failed. This is why `firestore.rules`
  needed no changes for this sprint — the order is always written once,
  fully formed, same as before.

The checkout flow (`AROM-Production/src/components/storefront/CheckoutSheet.tsx`)
calls two TanStack Start server functions in
`AROM-Production/src/lib/payments/pawapay.ts` — `initiatePawapayDeposit`
and `checkPawapayDepositStatus` — which currently stub PawaPay's real v2
API shape rather than calling `api.sandbox.pawapay.io`, since no
credentials exist yet. The client polls `checkPawapayDepositStatus` a
few times with a delay between calls (client-side polling, not a
webhook — see [flows.md](flows.md#payment--pawapay-mobile-money) for why).
Swapping the stub bodies for real `fetch()` calls with a server-only
Bearer token is the only change needed once credentials exist; nothing
about the client flow or the `orders.payment` shape changes.

## Staff poste & per-person bonus tracking (sprint 17)

A staff account's `poste` is one of `"Directeur de Production"`,
`"Chargée de Commercialisation"`, or `"Personnalisé"` — set at invite
time (auto-fills the matching `menus`) or afterward from the
dashboard's "Équipe (staff)" card.

**This is two things, not one.** `poste` is a cosmetic label and a UI
`menus` preset — but `firestore.rules` also reads it directly to scope
data access per collection:

- `"Directeur de Production"` → read/write on `producteurs`,
  `approvisionnements`, `productions`, `stockMP`. Denied on
  `clients`/`ventes`/`marketing`/`products`/`orders`/`charges`.
- `"Chargée de Commercialisation"` → read/write on `clients`, `ventes`,
  `marketing`, `products`, `orders`. Denied on the production
  collections and `charges`.
- No `poste` set, or `"Personnalisé"` → unchanged from before this
  sprint: full read/write on every internal collection, `menus` is
  UI-only. This is deliberate, not a gap — see below.

**Opt-in, never a migration.** Every staff account that existed before
this sprint keeps exactly the access it had — assigning a specific
named poste is something admin does per person, on purpose. This is
why there's no "everyone gets scoped" cutover: an account only becomes
data-scoped the moment admin picks a poste for it.

**`staffUid` on `productions`/`ventes`.** Entering a production lot or
a sale no longer asks for a `responsable`/`commerciale` name to type —
it's auto-filled from the logged-in account, and a `staffUid` is
stamped alongside it. The dashboard's per-person bonus cards filter
`productions`/`ventes` by `staffUid` to compute each person's own
prime/commission; anything with no matching `staffUid` (pre-sprint-17
data, or logged by an unscoped/admin account) shows as a "Non attribué"
line so the org-wide total still reconciles instead of quietly
excluding it.

See [rbac.md](rbac.md#poste-based-data-scoping-sprint-17) for the exact
`firestore.rules` shape and how it was verified.

## Order → ventes bridge

Marking an order "Livrée" in the dashboard's Commercialisation section
(`OrdersCard` in `AROM-Production/src/routes/dashboard.tsx`) atomically:

1. Sets the order's `status` to `fulfilled`.
2. Writes one `ventes/{id}` row per order line item, via `writeBatch` so
   both happen together or not at all.

Each generated `ventes` doc uses a deterministic ID
(`VTE-ORD-<orderId>-<itemIndex>`) rather than a random one, so re-running
the conversion (e.g. a retried click) overwrites the same doc instead of
duplicating it — `setDoc`, not `addDoc`. Conventions for the generated
row: `canal` is always `"Grossiste"` (distinguishes partner/storefront
sales from the channels used by manually-entered ventes), `commerciale`
is `"Boutique partenaire"`. As of [sprints/08](<sprints/[DONE] 08-pawapay-payment-stub.md>),
`encaisse` reflects the order's `payment`: the full line amount when
`payment` is set (a PawaPay deposit is only ever `completed` by the time
the order exists at all, and cash on delivery is collected at the exact
moment the order is marked "livrée"), or `0` for pre-sprint-08 orders
that predate the `payment` field, where nothing can be assumed about
whether cash changed hands.

## What's intentionally not modeled yet

- No editable delivery date after confirm (sprint 07) — set once at
  confirm time, no later edit path.
- No multiple product photos or structured attributes (sprint 14) — one
  photo, one free-text description.
