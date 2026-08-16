# Sprint 23 — Automated FC→USD currency exchange + Promotion as its own sidebar item

**Status:** Done — [Teddmab/AROM-Production#19](https://github.com/Teddmab/AROM-Production/pull/19)

**Rôle concerné :** Admin / Staff
**Page / zone :** Dashboard — header, Exécutif, Finances, KPI stratégiques,
Approvisionnement, Production, Stock, Commercialisation, Marketing, Parcours
production, sidebar

## Pourquoi maintenant

Two client requests delivered together:

1. "Let's integrate currency exchange automated, so we can pull the value
   in USD without having to make the calculation, or manually check."
   Every monetary figure in the ERP is FC-only; converting to USD to
   sanity-check a number against an external reference meant leaving the
   app and doing the division by hand.
2. "Let's have Promotion as a separate menu under Parcours production."
   Sprint 22 pulled Approvisionnement/Production/Stock/Commercialisation
   out of the top-level sidebar into `subItems` of "Parcours production"
   for being redundant with that page. "Promotion" — the storefront promo
   banner editor, previously a tab inside Commercialisation — gets the
   same treatment: its own reachable page, not buried a click deeper
   inside another section's tabs.

## Dans le périmètre

- **Cached FC/USD rate, not a live call per render.** `ExchangeRate-API`
  (`open.er-api.com`) is free, needs no API key, and supports CDF — but
  its own guidance is to poll at most hourly, ideally daily. `ErpProvider`
  (`store.tsx`) subscribes to a new singleton doc, `config/exchangeRate`
  (`fcPerUsd: number`, `fetchedAt: string`), and — only when the cached
  value is missing or more than 24h old — fetches a fresh rate and writes
  it back, so every open dashboard tab shares one fetch instead of each
  hitting the API independently.
- **Admin-only write, accepted as-is — no `firestore.rules` change.**
  `config/exchangeRate` falls under the existing generic `config/{doc}`
  rule (admin/staff read, admin-only write) — the same rule every other
  config doc except `config/promo` already follows. A staff session's
  background refresh attempt is simply caught and ignored client-side; the
  cached rate just waits one more day for an admin session to refresh it.
  Not worth a rules carve-out for a low-stakes, non-sensitive, infrequent
  background write.
- **`usdFormat(fcAmount, fcPerUsd)`** (new, `model.ts`) — returns
  `≈ X $` or `null` when no rate is cached yet, so every call site can
  simply omit the USD line rather than show a wrong or zeroed conversion.
- **Wired into summary figures only — confirmed directly with the
  client**, not into individual table rows: the header's "CA encaissé,"
  every monetary `KpiTile` across the dashboard (Exécutif, Approvisionnement,
  Production, Stock, Commercialisation, Marketing, KPI stratégiques), the
  Exécutif "Synthèse financière" table, all three Finances tables
  (Revenus/Coûts/Résultat), and Parcours production's Stock/Commercialisation
  funnel-stage secondary figures. Sales/purchase journals and other
  row-level tables stay FC-only, unchanged.
- **Promotion pulled out of Commercialisation's tabs into its own
  sidebar item.** New `PromotionSection` (wraps the existing `PromoCard`
  unchanged) added as a 5th, `isSection: true` member of "Parcours
  production"'s `subItems`, alongside Approvisionnement/Production/Stock/
  Commercialisation — same `alwaysExpanded` sidebar treatment sprint 22
  gave the other four. Commercialisation's own tabs drop from 4 to 3
  (Catalogue / Commandes / Ventes & clients).
- **Fixed the same class of regression sprint 22 hit.** `ALL_MENU_OPTIONS`
  (the flattened list `InviteCard`'s "Personnalisé" poste picker reads
  from) picks up "Promotion" automatically since it derives from
  `SECTIONS`' `subItems`. Separately, "Chargée de Commercialisation"'s
  poste preset `menus` (`src/lib/firebase/auth.tsx`) gained `"promotion"`
  — without this, a poste-scoped Commercial staff account would have lost
  access to the promo editor it previously reached via the
  "commercialisation" menu entry's own tab.

## Hors périmètre

- A `firestore.rules` carve-out letting staff sessions write
  `config/exchangeRate` directly — the admin-only fallback is simple and
  sufficient given AROM has admin activity most days; revisit only if a
  campaign goes admin-inactive for multiple days and the rate visibly
  goes stale.
- USD on every table row (sales journal, purchase receipts, etc.) — the
  client explicitly chose "summary figures only" over "everywhere money
  appears" when asked directly.
- Any change to `config/promo`'s own write permissions (already
  admin-only, pre-existing, unrelated to this move).

## Décisions déjà actées

- **Summary figures only, not every FC amount.** Asked directly; the
  client picked showing USD on header/KPI-tile/summary-table figures
  over adding it to every individual sales/purchase row, which would
  have doubled the width of several already-dense tables.
- **No `firestore.rules` change for the exchange-rate cache.** Matches
  the project's established preference for staying within the existing
  admin-write/staff-read pattern rather than adding a rules carve-out for
  a background convenience feature.
- **Promotion nests under Parcours production, not a standalone top-level
  item** — same reasoning as sprint 22: it's presentational sidebar
  grouping only (`isSection: true` in `subItems`), so no new nesting
  level and no change to how the page itself renders.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` or
`AROM-Backend` change. The exchange-rate fetch is a plain client-side
`fetch()` to a public, keyless third-party API — no server-side proxy,
consistent with the project's deliberate no-Cloud-Functions stance
(roadmap #10).

## Livrable

`AROM-Production`: `src/lib/erp/model.ts` (`usdFormat`), `src/lib/erp/store.tsx`
(`config/exchangeRate` subscribe/fetch/cache, `fcPerUsd` on `useErp()`),
`src/routes/dashboard.tsx` (`PromotionSection`, `SECTIONS`/`SectionId`
update, `KpiTile` `secondary` prop, USD wiring across 7 sections),
`src/lib/firebase/auth.tsx` (`STAFF_POSTES` menu update). `AROM-Documentation`
(this file, `data-model.md`).

## Test de fumée

- [x] Header "CA encaissé" shows `≈ X $` under the FC figure once a rate
      is cached
- [x] Exécutif's "Synthèse financière" and all three Finances tables show
      the USD equivalent alongside each FC amount
- [x] Individual table rows (sales journal, purchase receipts, etc.)
      remain FC-only
- [x] "Promotion" is reachable directly from the sidebar under "Parcours
      production," independent of Commercialisation
- [x] Commercialisation now shows 3 tabs (Catalogue / Commandes /
      Ventes & clients), not 4
- [x] The "Personnalisé" poste picker in Invitations offers "Promotion"
      as its own pill
- [x] Editing/activating a promo from its new page still shows live on
      the storefront

Covered by the automated regression suite (`node regression-suite.mjs`,
55/55 passing, two consecutive clean runs).
