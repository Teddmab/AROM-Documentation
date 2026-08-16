# Sprint 21 — Real-number calculation breakdowns in aggregate-value modals

**Status:** Done — [Teddmab/AROM-Production#17](https://github.com/Teddmab/AROM-Production/pull/17)

**Rôle concerné :** Admin / Staff (Direction Générale principalement)
**Page / zone :** Dashboard — Exécutif, Finances, KPI stratégiques, Approvisionnement,
Production, Stock, Commercialisation, Parcours production

## Pourquoi maintenant

Direct follow-up to sprint 19's record detail modals: "aggregated
values in a card such as Résultat brut, when we click to open the
modal, we need to show how we calculated the value... for each and
every card that has an aggregated value... showing the data from
Approvisionnement, Production, etc. all the way to how we reach that
value." Sprint 19 gave every computed field a one-line *formula*
description ("Calculé automatiquement : chiffre d'affaires − total
des coûts") — accurate, but not what was asked: an actual trace with
the campaign's real current numbers, not just the abstract relationship.

## Dans le périmètre

- **`buildBreakdowns(computed)`** (new, `dashboard.tsx`) — one shared
  function building the real, current-number calculation trail for
  every derived figure in `engine.ts`'s `computeErp()`, called once
  per section render and reused everywhere that figure is shown, so
  the same number never gets two different explanations:

  | Figure | Trail |
  | --- | --- |
  | Total coûts | Achats ananas + Transport + Autres charges + Marketing = Total coûts |
  | Résultat brut | Chiffre d'affaires − Total coûts = Résultat brut |
  | Marge brute | Résultat brut ÷ Chiffre d'affaires = Marge brute |
  | Rendement sur coûts | Résultat brut ÷ Total coûts |
  | Coût moyen / bouteille | Total coûts ÷ Bouteilles produites |
  | Prix moyen vendu | Chiffre d'affaires ÷ Bouteilles vendues |
  | Marge unitaire | Prix moyen vendu − Coût moyen / bouteille |
  | Taux d'encaissement | Encaissements ÷ Chiffre d'affaires |
  | Créances clients | Chiffre d'affaires − Encaissements |
  | Rendement volume | Volume conditionné ÷ Volume de jus |
  | Taux de pertes | Pertes ÷ Volume de jus |
  | Taux de transformation | Kg transformés ÷ Kg achetés |
  | Taux de vente | Bouteilles vendues ÷ Bouteilles produites |
  | Stock actuel | Bouteilles produites − Bouteilles vendues |

  Each intermediate line is labeled with the module it comes from
  ("(Approvisionnement)", "(Production)", "(Commercialisation)") so
  the trail visibly crosses module boundaries, not just formula
  algebra within one section.
- **`DetailField`** (`RecordDetailModal.tsx`) gained an optional
  `breakdown?: { label: string; value: string }[]` — an ordered list
  rendered as a small bordered table under the field's description,
  last row emphasized (bold, tinted background) as the "= final
  value" line.
- **`KpiTile`** and the Parcours production **`FunnelStage`**
  (sprint 20) both gained a matching `breakdown` prop, passed straight
  through into the same `RecordDetailModal` they already open on
  click — one rendering, three entry points.
- **Wired everywhere the figure already appeared**: Exécutif's
  "Synthèse financière" table and its Résultat brut/Marge brute KPI
  tiles; all three Finances tables (Revenus/Coûts/Résultat); KPI
  stratégiques' full indicator list; Production's Rendement/Pertes
  tiles; Commercialisation's Taux d'encaissement/Créances tiles;
  Stock's three top tiles and its per-format "Stock produits finis"
  modal; and all four Parcours production funnel stages.

## Hors périmètre

- ROI marketing's breakdown — the raw per-action `ventesGenerees` sum
  isn't exposed on `ErpComputed` (only folded into the final ratio
  internally), so a real trail would need either a new computed field
  or recomputing from `state.marketing` in the component. Left with
  its sprint-19 one-line description for now; small, easy follow-up
  if wanted.
- Personnel's per-poste objectif tables and per-person prime/commission
  tables — these are facsimiles of figures already fully explained on
  their primary pages (Production/Commercialisation), so weren't
  duplicated here to keep this sprint's scope bounded.

## Décisions déjà actées

- **One shared `buildBreakdowns()`, not per-section duplicates.** Every
  section that shows Total coûts, Résultat brut, etc. already reads
  from the same `useErp()` `computed` object — building the breakdown
  once per render and passing it down guarantees Exécutif, Finances,
  and KPI stratégiques can never show a different trail for the same
  number.
- **Last breakdown row is always the stated final value**, visually
  distinguished (bold, tinted) — a consistent convention so users
  learn to read "the last line is the answer" once, everywhere.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` or
`AROM-Backend` change; purely a richer read of numbers `computeErp()`
already produces.

## Livrable

`AROM-Production`: `src/components/erp/RecordDetailModal.tsx`
(`breakdown` field + rendering), `src/routes/dashboard.tsx`
(`buildBreakdowns()`, `KpiTile`/`FunnelStage` `breakdown` prop,
wiring across 7 sections). `AROM-Documentation` (this file).

## Test de fumée

- [x] Clicking "Résultat brut" (Finances or Exécutif) shows Chiffre
      d'affaires, Total coûts, and Résultat brut as three lines with
      real current numbers, the last one emphasized
- [x] The same numbers shown in the breakdown match the adjacent
      Finances tables exactly
- [x] Clicking "Marge brute," "Taux d'encaissement," "Rendement
      volume," etc. each show their own accurate trail
- [x] A Parcours production stage's headline number opens the same
      style of breakdown

Covered by the automated regression suite (`node regression-suite.mjs`,
47/47 passing, two consecutive clean runs).
