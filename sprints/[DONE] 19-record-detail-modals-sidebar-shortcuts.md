# Sprint 19 — Record detail modals + sidebar sub-nav shortcuts

**Status:** Done — [Teddmab/AROM-Production#15](https://github.com/Teddmab/AROM-Production/pull/15)

**Rôle concerné :** Admin / Staff (toutes les pages du dashboard)
**Page / zone :** Dashboard — les 24 tableaux répartis sur les 11 sections, plus la barre latérale

## Pourquoi maintenant

Direct follow-up to [sprint 18](<[DONE] 18-dashboard-ia-data-rethink.md>). Two explicit asks after
that sprint shipped:

1. "In each card that has items, when we click on the item, it should
   open a modal that shows all the related records, and what this
   info is for, where collected, etc., and option to delete, edit,
   etc." Every table row in the dashboard was inert — sprint 18's
   `Table` component (`dashboard.tsx`) rendered rows with no click
   handler, no way to see a field's meaning/provenance, and only a
   handful of tables even had a delete button (none had edit).
2. "I still believe some of the menu should have sub-menu... I would
   like to have a shortcut access from the menu." Sprint 18 added tabs
   inside Primes & personnel and Commercialisation, but reaching a
   specific tab still meant clicking the section, then clicking the
   tab inside the page. Confirmed via a clarifying question: auto-expand
   a section's tabs in the sidebar only once it's active, not a
   permanently-long sidebar.

## Dans le périmètre

- **Every table row opens a detail modal.** A structural inventory
  found 24 `Table` usages across the dashboard, split into two groups:
  - **Record-backed (11 tables)** — Appro réceptions & producteurs,
    Production lots, Stock mouvements, Commandes boutique partenaires,
    Ventes, Marketing actions, Charges fixes, invites, Boutiques
    partenaires, Équipe (staff). 8 of these get full edit + delete in
    the modal (Appro réceptions, Registre des producteurs, Production
    lots, Stock mouvements, Ventes, Marketing actions, Charges fixes,
    plus Boutiques partenaires' contact fields). Invites get delete
    (revoke) only — editing an accepted-or-not invite's fields after
    creation doesn't make sense against how `firestore.rules` validates
    redemption. Commandes and Équipe stay explanation-only in the
    modal — their existing inline actions (Confirmer/Annuler/Marquer
    livrée; the poste `<select>`) remain the primary/only way to
    change them, so there's exactly one path per action instead of two
    competing ones.
  - **Computed/aggregate (13 tables)** — Exécutif's two summary
    tables, Stock produits finis, Commercial "Portefeuille clients par
    canal", Finances' Revenus/Coûts/Résultat, Personnel's per-poste
    objectifs/prime-per-person tables and "Total primes campagne". No
    single doc backs a row here, so the modal opens in
    explanation-only mode — no Modifier/Supprimer.
- **`KpiTile` gets the same treatment.** All 24 usages across the
  dashboard now carry a `description` and open the same modal on
  click, explaining what the figure is and where it's computed from.
- **New reusable `RecordDetailModal`**
  (`src/components/erp/RecordDetailModal.tsx`) — follows the existing
  hand-rolled bottom-sheet convention already used by the storefront's
  `OrderDetailSheet`/`ProductDetailSheet` (fixed backdrop +
  `stopPropagation` panel + X close), not the unused shadcn
  Dialog/Sheet primitives. One overlay convention app-wide.
- **Sidebar shortcuts.** `SECTIONS` gained an optional `subItems` list
  on the two tabbed sections; once a tabbed section is active, its
  tabs appear as an indented list directly under it in the sidebar,
  each a direct jump. Tab state moved from each section's own
  uncontrolled `<Tabs defaultValue>` to a controlled
  `activeSubTab` map lifted into `Dashboard()` — Radix `Tabs` supports
  controlled mode natively, so this was prop plumbing, not a Tabs
  rework. A side benefit: tab selection now survives navigating away
  and back to the same section within one page load (previously reset
  to the first tab every time, since state was local to the
  unmounted/remounted section component).

## Hors périmètre

- Editing Commandes or Équipe (staff) directly through the modal —
  both already have a purpose-built inline flow (order status
  transitions; poste assignment) that the modal would otherwise
  duplicate. The modal still explains every field on those two.
- A document-upload KYC review workflow — unrelated to this sprint
  (see sprint 18); Boutiques partenaires' modal only adds the ability
  to fix a contact name/phone typo, which admin already had write
  access to via the same `updateDoc(users/{uid}, ...)` pattern
  `StaffCard` established in sprint 17.
- A literal cross-collection "journey" view tracing one physical batch
  from Approvisionnement through Production, Stock, and
  Commercialisation — raised in the same conversation as a follow-on
  idea, but the current data model has no linking fields between those
  collections (a `Production` doc doesn't reference the
  `Approvisionnement` receipt it consumed, etc.), so it needs its own
  design pass, not a bolt-on here. Tracked as a next step, not started.

## Décisions déjà actées

- **Additive, not a redesign.** Every existing inline action (Vérifié
  toggle, poste `<select>`, invite copy/revoke, order status buttons,
  every pre-existing Suppr. button) was kept exactly as it worked
  before — each just gained `stopPropagation()` so it doesn't also
  trigger the row's new click-to-open behavior. Lowest-regression-risk
  way to touch this much surface area in one sprint.
- **Explanation content is real content, not boilerplate.** Every
  field's description says specifically what it is and, for computed
  fields, exactly how it's derived (e.g. "Calculé automatiquement :
  quantité reçue × prix/kg") — sourced from `data-model.md` and
  `engine.ts`, not guessed.
- **Auto-expand-when-active for the sidebar**, not a permanently
  nested tree — confirmed directly with the client over a permanently-
  long-sidebar alternative. Keeps the 9 non-tabbed sections exactly as
  compact as before.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` or
`AROM-Backend` change needed; every write in this sprint reuses a
collection/pattern already writable by admin (`addRow`/`removeRow`
from `useErp()`, or a direct `updateDoc` matching the precedent set by
`StaffCard`/`BoutiquesCard`/`PromoCard` in earlier sprints).

## Livrable

`AROM-Production`: new `src/components/erp/RecordDetailModal.tsx`;
`src/routes/dashboard.tsx` (onRowClick wiring across all 24 tables,
`KpiTile` description prop, sidebar `subItems` + controlled tab
state). `AROM-Documentation` (this file).

## Test de fumée

- [x] Clicking a record-backed row (e.g. an Appro réception) opens a
      modal with matching field values and their explanations
- [x] Editing a field via "Modifier" → "Enregistrer" persists
- [x] Deleting a record via "Supprimer" → confirm removes the row
- [x] Clicking a computed/aggregate row (e.g. Exécutif's synthèse
      table) opens a modal with no Modifier/Supprimer
- [x] Clicking a KPI tile with a description opens the same
      explanation modal
- [x] From Primes & personnel, the sidebar shows its 5 tabs indented
      underneath it; clicking "Boutiques partenaires" there jumps
      straight to that tab's content
- [x] Every pre-existing inline action (Vérifié toggle, poste select,
      invite revoke, order status buttons, existing Suppr. buttons)
      still works unchanged

Covered by the automated regression suite (`node regression-suite.mjs`,
42/42 passing, two consecutive clean runs).
