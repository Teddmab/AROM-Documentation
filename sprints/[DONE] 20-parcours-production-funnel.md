# Sprint 20 — Parcours production funnel view + responsive modals

**Status:** Done — [Teddmab/AROM-Production#16](https://github.com/Teddmab/AROM-Production/pull/16)

**Rôle concerné :** Admin / Staff (Direction Générale principalement)
**Page / zone :** Dashboard — nouvelle section "Parcours production" ; tous les modaux de l'app

## Pourquoi maintenant

Raised twice in the same session as sprint 19, refined into a concrete
ask: "always have a look on the full journey of the product" across
Approvisionnement, Production, Stock, and Commercialisation. Two
design forks were resolved directly with the client via clarifying
questions before any code was written:

1. **Aggregate funnel, not literal per-batch traceability.** AROM
   pools raw ananas in shared storage rather than keeping each
   delivery physically segregated, and `Production`/`stockMP`/`Vente`
   records have no linking fields to each other today. Building
   literal batch genealogy would mean new required fields at
   data-entry time for non-technical staff, for a distinction the
   physical process doesn't actually preserve. Confirmed: a
   campaign-wide funnel using numbers that already exist is what's
   wanted (see roadmap #18 for the traceability idea, still
   deliberately unstarted).
2. **A new 5th section, not a merge.** Approvisionnement, Production,
   Stock, and Commercialisation stay exactly as they are —
   Commercialisation keeps its sprint-18 tabs unchanged. One new
   sidebar item, not a restructuring of the existing four.

Separately, mid-session: "the modals we are currently using are too
mobile first... when on laptop, the modal is a normal modal, but when
on mobile, we keep the current version." All four hand-rolled
modal/sheet components in the app were bottom-sheets at every viewport
width.

## Dans le périmètre

- **`ParcoursSection`** (new, `dashboard.tsx`) — four `FunnelStage`
  cards connected by arrows, built entirely from `useErp()` values
  that already exist:

  | Stage | Headline | Secondary |
  | --- | --- | --- |
  | Approvisionnement | `computed.kgAchetes` kg reçus | nombre de réceptions |
  | Production | `computed.bouteillesProduites` bt | taux de transformation (kg transformé / kg reçu) |
  | Stock | somme de `computed.stockPF[].stock` bt | valeur du stock produits finis |
  | Commercialisation | `computed.bouteillesVendues` bt | CA, taux vendu sur produit |

  Each card's "Voir le détail →" jumps straight to that section via a
  new `onNavigate: (id: SectionId) => void` prop from `Dashboard()` —
  same prop-lifting pattern sprint 19 used for `activeTab`/`onTabChange`.
  Ratios divide-by-zero-guard to `"—"` instead of `NaN%`, so an
  early-campaign state with no production yet reads as an honest "not
  started."
- **Sections renumbered** to make room: Marketing 05→06, Finances
  06→07, Primes & personnel 07→08, KPI stratégiques 08→09, Paramètres
  ERP 09→10, Feuille de route 10→11. Each section's own `eyebrow`
  badge was updated to match — the sidebar badge and the page's own
  header badge never disagreed before this sprint, kept that way.
- **`export.ts`** gained `"parcours"` in `ExportSection`, a
  `SECTION_LABELS` entry, and a `buildReport()` block — Parcours is
  exportable to PDF/Excel like every other top-level section.
- **Responsive modals.** `RecordDetailModal`, `OrderDetailSheet`,
  `ProductDetailSheet`, and `CheckoutSheet` all share the identical
  hand-rolled overlay classes (confirmed by grep before touching
  anything — same convention, copy-pasted four times). All four got
  the same two-class addition: the backdrop gains `sm:items-center
  sm:p-4`, the panel gains `sm:rounded-3xl sm:pb-0`. Below the `sm`
  breakpoint (640px), behavior is pixel-identical to before — full-width,
  glued to the bottom, only top corners rounded. At `sm:` and up, it's
  a centered, fully-rounded, backdrop-padded dialog.

## Hors périmètre

- Literal per-batch/lot traceability (roadmap #18) — this sprint
  answers "journey" with the aggregate-funnel design the client
  confirmed, not a data-model change.
- Merging Approvisionnement/Production/Stock/Commercialisation into
  one menu — considered and explicitly rejected in favor of a 5th,
  additive section.
- Adding "Parcours production" to a named poste's default menu — see
  Décisions déjà actées below.

## Décisions déjà actées

- **Not added to `STAFF_POSTES` presets.** A "Directeur de Production"
  poste-scoped account is denied `ventes` read access at the
  `firestore.rules` level (sprint 17) — Parcours needs to read across
  all four domains (including `ventes` for the Commercialisation
  stage), so adding it to that poste's menu would surface a permission
  error for that account. Only admin and unscoped/"Personnalisé" staff
  (already full-access) see it. Pure UI-visibility decision via
  `menus`, no `firestore.rules` change.
- **`sm:` (640px) as the mobile/desktop modal cutoff** — matches
  Tailwind's own convention for "no longer a narrow phone"; a tablet
  in landscape or wider already gets the centered-dialog treatment.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` or
`AROM-Backend` change for either half of this sprint.

## Livrable

`AROM-Production`: `src/routes/dashboard.tsx` (`ParcoursSection`,
`FunnelStage`, `FunnelArrow`, `SECTIONS` renumbering,
`onNavigate` wiring), `src/lib/erp/export.ts` (parcours export case),
`src/components/erp/RecordDetailModal.tsx` +
`src/components/storefront/{OrderDetailSheet,ProductDetailSheet,CheckoutSheet}.tsx`
(responsive overlay classes). `AROM-Documentation` (this file).

## Test de fumée

- [x] Parcours production shows all four stages with real figures
- [x] The Approvisionnement stage's kg-reçus figure matches the
      Approvisionnement page's own "Quantité reçue" KPI tile exactly
- [x] "Voir le détail" on a stage card navigates to that section
- [x] A poste-scoped "Directeur de Production" test account's sidebar
      does not show "Parcours production"
- [x] On a desktop viewport, the record detail modal is a centered,
      fully-rounded dialog, not a full-width bottom sheet
- [x] On a mobile viewport, every modal is unchanged — full-width,
      bottom-anchored, top corners only

Covered by the automated regression suite (`node regression-suite.mjs`,
46/46 passing, two consecutive clean runs).
