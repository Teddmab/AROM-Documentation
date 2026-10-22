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
| `config/parametres` | `objectifAnanasKg`, `objectifBouteilles`, `prix500/330/300`, `tauxCommission`, `tauxPrimeProduction`, `tauxPertesMax`, `distanceFournisseurKm`, `debutProduction`, `finProduction`, `finCommercialisation`, `objectifClients`, `objectifMargeBrute`, `rendementAttenduLPour100Kg?`, `ecartAcceptablePct?` | Single doc, edited from the "Paramètres ERP" dashboard section, and (the last two fields) from the mobile app's `/admin-settings/production` screen. `rendementAttenduLPour100Kg`/`ecartAcceptablePct` (production-settings integration) are mobile-only additions — undefined until an admin sets them at least once; see automation-engine.md's settings-storage decision. `tauxPertesMax` is shared: the web dashboard's own KPI formulas and the mobile settings screen (labeled there "Perte maximale acceptable") read/write the same field, deliberately not duplicated. |
| `producteurs/{id}` | `nom`, `village`, `secteur`, `territoire`, `telephone`, `produit`, `capaciteKgMois`, `prixConvenu`, `statut`, `observations`, `photoUrl?`, `createdByUid?`, `createdByPoste?`, `createdAt?` | Supplier registry. `photoUrl`/`createdByUid`/`createdByPoste`/`createdAt` (MOB-04B) are optional — set by the mobile app's agent-created-producer flow (Storage download URL under `producteurs/{id}/`, and the creating account's uid/poste/timestamp), absent on producers seeded or created before this sprint. **Known gap**: there is no verification workflow — `statut` is free text (e.g. `"Actif"`, the only convention in use) with no `verified`/`pending` field backing it; a producer created from the field is not distinguishable server-side from an admin-entered one, and the mobile app's "en attente de vérification" messaging is local-only UI copy, not a real state. An admin-verification sprint would need to add that field deliberately rather than assume `statut` already covers it. |
| `approvisionnements/{id}` | `numero`, `date`, `idProducteur`, `fournisseur`, `village`, `produit`, `qteCommandeeKg`, `qteRecueKg`, `prixKg`, `transport`, `autresFrais`, `qualite`, `photoUrl?` | Purchase receipts. `photoUrl` (MOB-04) is a Storage download URL under `approvisionnements/{id}/`, set by the mobile app once a reception syncs — optional, absent on records created without a photo or from the web dashboard. |
| `productions/{id}` | `lot`, `date`, `kgUtilises`, `volumeJusL`, `q500`, `q330`, `q300`, `rejets`, `responsable`, `staffUid?`, `statut`, `parametresSnapshot?`, `expectedVolumeJusL?`, `ecartExplication?` | Production batches. `responsable` and `staffUid` (sprint 17) are auto-filled from the logged-in staff account, not typed — `staffUid` powers per-person bonus tracking; entries predating sprint 17 have no `staffUid`. `parametresSnapshot` (production-settings integration, mobile-only) is a map — `{rendementAttenduLPour100Kg?, ecartAcceptablePct?, tauxPertesMax?, snapshotAt}` — an **immutable copy** of `config/parametres`' automation-relevant fields, taken once when the mobile wizard's draft was created and never refreshed afterward, so a completed production's own "was this unusual?" interpretation never depends on the later-mutable settings doc (changing ADMIN settings mid-draft, or after submission, can't retroactively alter it). `expectedVolumeJusL` is AROM's own suggested output at the moment the worker confirmed the actual `volumeJusL` (`kgUtilises × parametresSnapshot.rendementAttenduLPour100Kg / 100`) — kept for review/audit, never the actual output itself. `ecartExplication` is the worker's own plain-language explanation, present only once `\|variance\| > parametresSnapshot.ecartAcceptablePct`. All three fields are absent on productions created before this addition, or when no yield standard was configured at draft-creation time — a fully legitimate, un-evaluated state, not an error. The ADMIN "Priorités" screen derives a live, deduplicating yield-variance priority from these fields (`AROM-Mobile`'s `adminPriorities.ts`) — no separate priority document, no manual resolved flag; see automation-engine.md. |
| `qualityControls/{id}` | `productionId`, `lot`, `date`, `echantillonL`, `couleurNormale`, `odeurNormale`, `bouteillePropre`, `quantiteConformeL`, `quantiteRejeteeL`, `decision` (`liberer`/`quarantaine`/`rejeter`), `motif?`, `commentaire?`, `photoUrl?`, `responsable`, `staffUid?` | Quality-control decisions on a production lot (MOB-05B, mobile-only — no web UI yet). One doc per control attempt, referencing `productions/{id}` via `productionId`; the mobile client enforces at most one active control per `productionId` (see AROM-Mobile's `qualitySync.ts`), not a Firestore rule. `photoUrl` is a Storage download URL under `qualityControls/{id}/`, optional. **Known gap**: releasing/quarantining/rejecting a lot here has no effect on any finished-goods stock field — none exists (`stockMP` below is raw material only) — so "available for sale" is UI-only copy derived from `decision`, not an enforced stock state. |
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
| `users/{uid}` | `email`, `displayName`, `role` (`admin`\|`staff`\|`partner`), `menus` (`"all"` or `string[]`), `active`, `createdAt`, `inviteId?`, `onboardingComplete?`, `contactName?`, `phone?`, `address? {ville, commune, quartier, repere?}`, `idNumber?`, `pointDeVente?`, `verified?`, `poste?` | `uid` matches the Firebase Auth UID. `inviteId` is only present on admin/staff accounts created via `/join` (see `invites` below); an audit trail, not read by any rule. The `onboardingComplete` through `verified` fields are partner-only. `onboardingComplete` through `pointDeVente` are written by the guided onboarding wizard — see below. `verified` (sprint 16) defaults `false` at signup and is flipped by admin from the dashboard's "Boutiques partenaires" card after a phone-call confirmation — informational only, never a gate on ordering. `idNumber` was write-only (collected at signup, never read back by the dashboard) until [sprint 18](<sprints/[DONE] 18-dashboard-ia-data-rethink.md>), which surfaces it next to the "Vérifié" toggle so admin has something to actually review before verifying. `poste` (sprint 17, staff-only, extended MOB-04) is `"Directeur de Production"` \| `"Chargée de Commercialisation"` \| `"Agent de collecte"` \| `"Personnalisé"` — see below |
| `invites/{id}` | `email`, `role` (`admin`\|`staff`), `menus` (`"all"` or `string[]`), `poste?`, `used`, `usedBy?`, `createdBy`, `createdAt` | Lets an admin grant admin/staff access without the CLI — see [rbac.md](rbac.md#how-adminstaff-accounts-are-provisioned). Single-use, email-locked; doc ID is the actual secret (get-by-ID is public, listing is admin-only). `poste` (sprint 17) carries through to the redeemed account's `users/{uid}` doc, validated the same way as `role`/`menus` — see below |
| `products/{id}` | `name`, `format`, `price`, `active`, `imageUrl?`, `description?` | Storefront catalog; seeded with the three bottle formats from `parametres`. `imageUrl` is optional (nullable) — populated by uploading a photo in the dashboard's Catalogue card, which stores it in Storage at `products/{id}/photo` and writes the resulting download URL here. Fully admin/staff-manageable from the dashboard as of [sprints/05](<sprints/[DONE] 05-admin-catalog-management.md>) — not just seed-script-only anymore. `description` (sprint 14) is free text shown on the storefront's product detail sheet — no structured attributes (ingredients, nutrition) yet |
| `orders/{id}` | `partnerId`, `partnerName`, `partnerPhone?`, `partnerAddress?`, `items: {productId, name, quantity, unitPrice, format}[]`, `total`, `status` (`pending`\|`confirmed`\|`fulfilled`\|`cancelled`), `createdAt`, `deliveryDate?`, `fulfilledAt?`, `payment: {method, status, ...}` | One doc per storefront order. `format` is a snapshot of the product's format at order time (not a live join), matching how `name`/`unitPrice` are already snapshotted. `partnerPhone`/`partnerAddress` are likewise a snapshot of the partner's profile at order time (a single formatted string for the address, not the structured object) — see below. `deliveryDate` (sprint 07) is set by hand by admin when confirming an order — ad-hoc, not a recurring schedule. `fulfilledAt` (sprint 07) is stamped automatically when an order is marked "livrée," so the storefront can show a real date instead of just the status. `payment` is written once at order creation and never mutated afterward — see below |
| `config/promo` | `active: boolean`, `headline: string`, `description?: string`, `productId?: string`, `startDate?: string`, `endDate?: string` | Single active-or-not storefront promo (sprint 06) — no rotation/multiple-banner queue. Shown on `/storefront` when `active` and (if set) within `startDate`/`endDate`. Admin-only write, but **signed-in read** (unlike `config/parametres`, which is admin/staff-only) — see [rbac.md](rbac.md) and the `config/promo` exact-path rule ahead of the `config/{doc}` wildcard in `firestore.rules` |
| `config/exchangeRate` | `fcPerUsd: number`, `fetchedAt: string` (ISO) | Cached FC/USD rate (sprint 23), fetched client-side from `open.er-api.com` (ExchangeRate-API, free tier, no key) whenever the cached value is more than 24h old. Falls under the generic `config/{doc}` rule — admin/staff read, **admin-only write** — so a stale rate simply waits for an admin session to refresh it; a staff session's refresh attempt is caught and ignored client-side, no `firestore.rules` change |

`firestore.indexes.json` defines a composite index on
`orders(partnerId ASC, createdAt DESC)` for the "my orders" query.

## Guided boutique onboarding (sprint 13)

`/storefront/signup` (`AROM-Production/src/routes/storefront/signup.tsx`)
is a 3-step wizard (reshaped from the original 5-step version for
OAuth-first signup — see [rbac.md](rbac.md#sign-in-methods)), so a
non-technical partner is walked through it one question at a time:

1. **Compte** — Google/Facebook OAuth (primary) or email/password
   (collapsed behind a secondary "Continuer avec e-mail" link) — creates
   only a Firebase Auth credential, no `users/{uid}` doc yet.
2. **Rôle** — "Boutique partenaire" (writes the `users/{uid}` doc with
   `role: "partner"`, `onboardingComplete: false`, via
   `createPartnerProfile`, then continues to step 3) or "J'ai une
   invitation d'équipe" (an inline sub-flow: paste an invite link/ID,
   preview it, confirm — see [rbac.md](rbac.md#sign-in-methods) for how
   that redeems an admin/staff invite for the already-authenticated
   account and skips step 3 entirely, landing on `/dashboard` instead).
3. **Coordonnées** (partner branch only) — one combined screen for
   `contactName`, `phone`, structured `address` (`ville` — dropdown of
   known Kasaï-region towns + "Autre" free text — `commune`, `quartier`,
   optional `repere` landmark, since DRC addresses commonly lack formal
   street addressing) and optional `idNumber` (CNI/RCCM, light-touch KYC —
   explicitly not a document-upload + admin-review flow, no Storage rules
   involved). Shows the serving depot, then writes everything in one
   `completePartnerOnboarding` call (`updateDoc` with
   `onboardingComplete: true` and `pointDeVente`).

**Two-write account creation, now split across steps 1 and 2 instead of
both inside step 1.** No `users/{uid}` doc exists until step 2 (role
chosen) — deliberately, since which doc shape to write (partner vs. an
invite's admin/staff role) isn't known until then. Step 3 is held in
local component state, not written per-field. This is still resumable if
abandoned: the route's own effect resumes at step 2 for a signed-in user
with no profile yet, or at step 3 for a partner profile with
`onboardingComplete: false` — `displayName`/`email` are re-populated from
the existing profile in the latter case. Note this relies on
`/storefront/signup` (like `/login` and `/join`) never being wrapped in
`RequireRole` — that component's "Accès indisponible" dead-end for a
no-profile signed-in user only applies to gated routes (`/dashboard`,
`/storefront`), not to the signup page itself, so a mid-flow, doc-less
account sitting on step 2 is an expected state, not an error.

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

## Staff poste & per-person bonus tracking (sprint 17, extended MOB-04)

A staff account's `poste` is one of `"Directeur de Production"`,
`"Chargée de Commercialisation"`, `"Agent de collecte"`, or
`"Personnalisé"` — set at invite time (auto-fills the matching `menus`)
or afterward from the dashboard's "Équipe (staff)" card.

**This is two things, not one.** `poste` is a cosmetic label and a UI
`menus` preset — but `firestore.rules` also reads it directly to scope
data access per collection:

- `"Directeur de Production"` → read/write on `producteurs`,
  `approvisionnements`, `productions`, `stockMP`. Denied on
  `clients`/`ventes`/`marketing`/`products`/`orders`/`charges`.
- `"Chargée de Commercialisation"` → read/write on `clients`, `ventes`,
  `marketing`, `products`, `orders`. Denied on the production
  collections and `charges`.
- `"Agent de collecte"` (MOB-04) → read/write on `producteurs`,
  `approvisionnements` only — narrower than `"Directeur de Production"`,
  no `productions`/`stockMP` access. Exists for the AROM-Mobile
  Agent-de-collecte role; has no per-person bonus card (no bonus
  formula is defined for receptions, unlike production/vente).
- No `poste` set, or `"Personnalisé"` → unchanged from before this
  sprint: full read/write on every internal collection, `menus` is
  UI-only. This is deliberate, not a gap — see below.

`producteurs`/`approvisionnements` also gained real field/type
validation on `create`/`update` as of MOB-04 (previously a blanket
`allow read, write` with no shape check — unlike `users`/`invites`).
See [rbac.md](rbac.md#poste-based-data-scoping-sprint-17-extended-mob-04)
for what that does and does not guarantee.

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

See [rbac.md](rbac.md#poste-based-data-scoping-sprint-17-extended-mob-04)
for the exact `firestore.rules` shape and how it was verified.

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

## Producer payables and Mombongo marketplace (MOB-06A - MOB-10, mobile-only)

These collections exist only for AROM-Mobile's admin-only "Factures"/"Marché
Mombongo" surfaces — there is no web dashboard UI for any of them yet.
Field/type shapes below mirror `AROM-Backend/firestore.rules`'s
`isValidProducerInvoice`/`isValidHarvestOffer`/`isValidHarvestInvoice`
functions exactly; see [rbac.md](rbac.md) for who can read/write each one.

| Collection | Fields | Notes |
| --- | --- | --- |
| `producerInvoices/{id}` | `idProducteur`, `approvisionnementId`, `devise`, `dateFacture`, `quantiteKg`, `prixUnitaireKg`, `transport`, `autresFrais`, `montantTotal`, `statut` (`recue`\|`a_verifier`\|`approuvee`\|`paiement_en_attente`\|`payee`\|`contestee`\|`annulee`), `referenceFournisseur?`, `dateEcheance?`, `motifContestation?`, `referenceExternePaiement?`, `mombongoInvoiceId?`, `mombongoCheckout?`, `createdByUid?`, `createdByPoste?`, `createdAt?`, `updatedByUid?`, `updatedAt?` | AROM-owned producer-payable domain (MOB-06A), deliberately provider-neutral — no Mombongo name/field is required by the schema itself, only optionally present once a payment is routed through them. `isAdmin()`-only read/write; immutable (`delete` is always `false`). `paiement_en_attente`/`payee`/`annulee` are unreachable from any admin write — only the dedicated Mombongo webhook identity can make those three transitions (MOB-07), never a manual "mark as paid." Single-tenant: no organization-scoping field exists anywhere in this schema. |
| `harvestOffers/{id}` | `listingId`, `mombongoOfferId`, `offerQuantityKg`, `offerPricePerKgCdf`, `status` (`pending`\|`won`), `createdAt`, `createdByUid`, `message?` | AROM's own record of an offer it made on Mombongo's farmer marketplace (Sprint DP, 2026-09-01) — AROM is the *buyer* here, a separate flow from `producerInvoices`' producer-submitted-invoice model. `isAdmin()`-only read; every write (including the initial `create`) goes through the Mombongo webhook identity, never a direct `isAdmin()` write — an admin action in the app calls a trusted AROM-Production server route, which writes as that identity. |
| `harvestInvoices/{id}` | `farmerId`, `listingId`, `amountUsd`, `quantityKg`, `commodity`, `statut` (`a_payer`\|`paiement_en_attente`\|`payee`), `createdAt`, `mombongoCheckout?` | A payable Mombongo itself originates once a harvest offer wins (pushed via Mombongo's `invoiceIssued` webhook event) — doc ID is Mombongo's own invoice ID, not an AROM-generated one. No `approuvee`/review step, unlike `producerInvoices`: there's no AROM-side réception to reconcile against, so it starts directly at `a_payer`. `isAdmin()` read, plus `isMombongoWebhook()` read/write; every write again goes through the webhook identity only. |
| `externalIntegrations/mombongo` | `baseUrl`, `partnerId`, `inboundSigningSecret`, `outboundVerifySecret`, `active` | Mombongo's own signing secrets (MOB-07) — set via `AROM-Backend/scripts/set-mombongo-integration-config.mjs`, never through the app. Read is `isMombongoWebhook()`-only, deliberately narrower than every other `isAdmin()`-readable collection in this file, so a secret has no path to a browser/app session. |

**What Mombongo actually populates, as of this pass**: `producerInvoices`
(status transitions only, via the webhook identity),
`harvestOffers`/`harvestInvoices` (its own marketplace/payables domain).
**Mombongo does not write to `products` or `orders`** — the storefront
catalog above stays a manually managed AROM collection. If AROM's
supply/product data is meant to sync from Mombongo (the "longer-term
vision" item in [roadmap.md](roadmap.md#from-the-2026-08-14-kickoff-meeting-client-ethical-mine)),
that is a distinct, unbuilt capability, not something already implied by
this integration.

**Decision record — external customer orders are not `producerInvoices`.**
`producerInvoices`/`harvestInvoices` model AROM *paying* a producer/farmer
(upstream). A future external-customer-order capability (an outside buyer
placing and paying for an order — downstream, and distinct from both the
`partner`/storefront `orders` collection above and from any
Mombongo-branded invoice) must be modeled as its own new domain —
`partnerOrders` or equivalent — never bolted onto `producerInvoices` or a
hypothetical `mombongoInvoices` collection. The two are opposite cash-flow
directions and have no shared schema need; conflating them would make a
`statut`/transition function ambiguous about which direction of money is
being described. See `docs/completion-plan/README.md` in AROM-Mobile for
the sprint (09) this gates.

## What's intentionally not modeled yet

- No editable delivery date after confirm (sprint 07) — set once at
  confirm time, no later edit path.
- No multiple product photos or structured attributes (sprint 14) — one
  photo, one free-text description.
