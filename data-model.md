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
| `productions/{id}` | `lot`, `date`, `kgUtilises`, `volumeJusL`, `q500`, `q330`, `q300`, `rejets`, `responsable`, `statut` | Production batches |
| `stockMP/{id}` | `date`, `produit`, `unite`, `type` (`Entrée`/`Sortie`/`Ajustement`), `entree`, `sortie`, `coutUnitaire`, `observation` | Raw-material stock ledger |
| `clients/{id}` | `nom`, `categorie`, `contact`, `zone`, `premierContact`, `statut` | Internal client registry (distinct from `users`/partner accounts) |
| `ventes/{id}` | `numero`, `date`, `idClient`, `client`, `canal`, `format`, `quantite`, `prixUnitaire`, `remise`, `encaisse`, `commerciale` | Sales journal |
| `marketing/{id}` | `numero`, `date`, `campagne`, `canal`, `cible`, `description`, `budget`, `coutReel`, `contacts`, `prospects`, `ventesGenerees` | Marketing actions |
| `charges/{id}` | `rubrique`, `budget`, `realise` | Fixed charges |

All derived KPIs (marge brute, rendement, primes, etc.) are computed
client-side from these by `AROM-Production/src/lib/erp/engine.ts` — none
of that is stored.

## Accounts & storefront

| Collection | Fields | Notes |
| --- | --- | --- |
| `users/{uid}` | `email`, `displayName`, `role` (`admin`\|`staff`\|`partner`), `menus` (`"all"` or `string[]`), `active`, `createdAt`, `inviteId?` | `uid` matches the Firebase Auth UID. `inviteId` is only present on admin/staff accounts created via `/join` (see `invites` below); an audit trail, not read by any rule |
| `invites/{id}` | `email`, `role` (`admin`\|`staff`), `menus` (`"all"` or `string[]`), `used`, `usedBy?`, `createdBy`, `createdAt` | Lets an admin grant admin/staff access without the CLI — see [rbac.md](rbac.md#how-adminstaff-accounts-are-provisioned). Single-use, email-locked; doc ID is the actual secret (get-by-ID is public, listing is admin-only) |
| `products/{id}` | `name`, `format`, `price`, `active`, `imageUrl?` | Storefront catalog; seeded with the three bottle formats from `parametres`. `imageUrl` is optional (nullable) — populated by uploading a photo in the dashboard's Catalogue card, which stores it in Storage at `products/{id}/photo` and writes the resulting download URL here. Fully admin/staff-manageable from the dashboard as of [sprints/05](sprints/05-admin-catalog-management.md) — not just seed-script-only anymore |
| `orders/{id}` | `partnerId`, `partnerName`, `items: {productId, name, quantity, unitPrice, format}[]`, `total`, `status` (`pending`\|`confirmed`\|`fulfilled`\|`cancelled`), `createdAt`, `payment: {method, status, ...}` | One doc per storefront order. `format` is a snapshot of the product's format at order time (not a live join), matching how `name`/`unitPrice` are already snapshotted. `payment` is written once at order creation and never mutated afterward — see below |

`firestore.indexes.json` defines a composite index on
`orders(partnerId ASC, createdAt DESC)` for the "my orders" query.

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
is `"Boutique partenaire"`. As of [sprints/08](sprints/08-pawapay-payment-stub.md),
`encaisse` reflects the order's `payment`: the full line amount when
`payment` is set (a PawaPay deposit is only ever `completed` by the time
the order exists at all, and cash on delivery is collected at the exact
moment the order is marked "livrée"), or `0` for pre-sprint-08 orders
that predate the `payment` field, where nothing can be assumed about
whether cash changed hands.

## What's intentionally not modeled yet

- No promo/banner doc yet — see
  [sprints/06](sprints/06-promo-banner.md) for the proposed
  `config/promo` shape.
