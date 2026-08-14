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
| `users/{uid}` | `email`, `displayName`, `role` (`admin`\|`staff`\|`partner`), `menus` (`"all"` or `string[]`), `active`, `createdAt` | `uid` matches the Firebase Auth UID |
| `products/{id}` | `name`, `format`, `price`, `active`, `imageUrl?` | Storefront catalog; seeded with the three bottle formats from `parametres`. `imageUrl` is optional (nullable) — populated by uploading a photo in the dashboard's Catalogue card, which stores it in Storage at `products/{id}/photo` and writes the resulting download URL here. Fully admin/staff-manageable from the dashboard as of [sprints/05](sprints/05-admin-catalog-management.md) — not just seed-script-only anymore |
| `orders/{id}` | `partnerId`, `partnerName`, `items: {productId, name, quantity, unitPrice, format}[]`, `total`, `status` (`pending`\|`confirmed`\|`fulfilled`\|`cancelled`), `createdAt` | One doc per storefront order. `format` is a snapshot of the product's format at order time (not a live join), matching how `name`/`unitPrice` are already snapshotted |

`firestore.indexes.json` defines a composite index on
`orders(partnerId ASC, createdAt DESC)` for the "my orders" query.

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
is `"Boutique partenaire"`, and `encaisse` is `0` — storefront orders
aren't a checkout (see [architecture.md](architecture.md)), so payment
status is reconciled manually afterward in the ventes journal like any
other credit sale.

## What's intentionally not modeled yet

- No promo/banner doc yet — see
  [sprints/06](sprints/06-promo-banner.md) for the proposed
  `config/promo` shape.
- No payment fields on `orders` yet — see
  [flows.md](flows.md#payment--pawapay-mobile-money) for the proposed
  `payment` sub-object.
