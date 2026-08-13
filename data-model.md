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
| `products/{id}` | `name`, `format`, `price`, `active` | Storefront catalog; seeded with the three bottle formats from `parametres` |
| `orders/{id}` | `partnerId`, `partnerName`, `items: {productId, name, quantity, unitPrice}[]`, `total`, `status` (`pending`\|`confirmed`\|`fulfilled`\|`cancelled`), `createdAt` | One doc per storefront order |

`firestore.indexes.json` defines a composite index on
`orders(partnerId ASC, createdAt DESC)` for the "my orders" query.

## What's intentionally not modeled yet

- No linkage between a storefront `orders` doc and the `ventes` collection
  — confirming/fulfilling an order today doesn't automatically create a
  `ventes` row. Bridging that is the natural next step once order
  fulfillment is a real workflow staff use daily (see
  [roadmap.md](roadmap.md)).
- No product photo field on `products` yet, though `storage.rules`
  already reserves `products/**` in Storage for this.
