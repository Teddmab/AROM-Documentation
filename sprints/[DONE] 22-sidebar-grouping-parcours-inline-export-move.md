# Sprint 22 — Sidebar grouping, inline Parcours details, export card repositioned

**Status:** Done — [Teddmab/AROM-Production#18](https://github.com/Teddmab/AROM-Production/pull/18)

**Rôle concerné :** Admin / Staff
**Page / zone :** Dashboard — sidebar, Parcours production, all 9 sections with an export card

## Pourquoi maintenant

Direct client feedback after seeing sprint 20 live, in three parts:

1. "Why do we have both Parcours production, and all Approvisionnement,
   Production, etc. — these should be as sub-menu, otherwise they are
   pretty redundant." Confirmed: having Parcours as a 5th standalone
   sidebar item next to the four sections it summarizes read as
   duplicated navigation, not a meaningful distinction.
2. "When we click 'voir le details' under a card, instead of showing a
   dedicated path, wouldn't it be more ergonomic to extend and show
   details under the 4 cards, and highlight the card that has been
   pressed?" Clicking a stage previously navigated clean away from the
   one page whose whole point is showing the connected picture.
3. "The export card should not be on top on the different pages — it
   is a bit misleading, makes it think like it's a filter." The
   Campagne/Du/Au + Export PDF/Excel card sat first on every page,
   reading as the page's primary control when it's a secondary
   reporting tool.

## Dans le périmètre

- **Sidebar grouping, not a real container.** Approvisionnement,
  Production, Stock, and Commercialisation moved from independent
  top-level `SECTIONS` entries into `subItems` of "Parcours
  production" with a new `alwaysExpanded: true` flag (unlike Personnel/
  Commercialisation's own in-page-tab subItems, which only show once
  their parent is active — these show permanently, since they're full
  pages people need to reach directly). Clicking one still does a
  plain `setActive(id)` to that exact same independent `SectionId` —
  Commercialisation's sprint-18 in-page tabs are entirely untouched,
  so this never creates a third nesting level.
- **Visibility preserved for poste-scoped accounts.** A "Directeur de
  Production" account has `appro`/`production`/`stock` in `menus` but
  not `parcours` itself (sprint 20's deliberate decision, since
  Parcours reads `ventes` which that poste is denied). `visibleSections`
  now shows the "Parcours production" group if the account can open
  *either* the group page *or* at least one member section, and each
  sub-item is independently filtered the same way — so that account
  still sees and can click Approvisionnement/Production/Stock, just
  with a non-clickable "Parcours production" label above them instead
  of the funnel page itself.
- **Fixed a real regression this surfaced.** `InviteCard`'s manual
  "Sections accessibles" picker (for a "Personnalisé" poste invite)
  built its pill list directly from `SECTIONS`, which no longer
  contains `appro`/`production`/`stock`/`commercialisation` as
  top-level entries. Added `ALL_MENU_OPTIONS` — `SECTIONS` flattened
  to include every `isSection` sub-item — so all 12 real sections stay
  individually pickable, not just the 8 top-level ones.
- **Inline expand instead of navigation.** `FunnelStage` gained
  `expanded`/`onToggle` (replacing `onView`), rendering a highlighted
  ring (`border-primary ring-2 ring-primary/30`) when its stage is the
  expanded one. `ParcoursSection` tracks one `expandedStage` at a time
  and renders a `Card` below the funnel row with a compact table of
  that stage's most recent records (last 5 réceptions/lots/ventes,
  sorted by date, or the Stock produits finis breakdown) plus an
  "Ouvrir {stage} en entier →" button — the only thing that still
  calls `onNavigate`.
- **Export card moved to the bottom.** All 9 `<ExportBar section="...">`
  usages (Exécutif, Approvisionnement, Production, Stock,
  Commercialisation, Parcours production, Marketing, Finances, KPI
  stratégiques) moved from directly under `SectionHeader` to just
  before each section's closing `</div>`. Same component, same
  fields, same shared campaign filter (sprint 18) — purely a position
  change.
- **Renumbering.** Remaining top-level sections shifted to match the
  new sidebar order: Parcours 01, Marketing 02, Finances 03, Primes &
  personnel 04, KPI stratégiques 05, Paramètres ERP 06, Feuille de
  route 07. Approvisionnement/Production/Stock/Commercialisation keep
  their original "Module ERP 01–04" page eyebrows unchanged (no longer
  shown as competing sidebar badges, so no mismatch risk). Parcours
  production uses a non-numbered "Vue transverse" eyebrow instead of
  claiming a "Module ERP 0X" slot, since — unlike Approvisionnement
  through Personnel — it isn't one of the original Excel-derived
  modules.

## Hors périmètre

- Any change to Commercialisation's own sprint-18 tabs (Catalogue/
  Promotions/Commandes/Ventes & clients) — completely untouched,
  which is exactly what avoids the 3-level nesting problem this
  design was chosen to solve.
- Splitting the Campagne/Du/Au filter from the Export PDF/Excel
  buttons into two separate controls — considered (see Décisions
  déjà actées) but not what was asked for.

## Décisions déjà actées

- **Grouping is presentational, never a real container**, specifically
  so poste-scoped visibility rules (sprint 17) and Commercialisation's
  own tabs (sprint 18) both keep working exactly as before — the
  entire fix is in how the sidebar *renders* `SECTIONS`, not in the
  underlying section components.
- **Move the export card as one unit, not split it apart.** Confirmed
  directly: keep Campagne/Du/Au and Export PDF/Excel together as the
  same card, just relocated — not two separate controls in two
  separate places.
- **Inline panel shows a *summary*, not the full editable table.** The
  "Ouvrir ... en entier" link exists precisely so anyone who wants to
  add/edit/delete records still goes to the real page with the full
  `EntryForm`/`RecordDetailModal` machinery — the inline panel is
  read-only, by design, to stay light.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` or
`AROM-Backend` change.

## Livrable

`AROM-Production`: `src/routes/dashboard.tsx` only (`SECTIONS`
restructuring, `ALL_MENU_OPTIONS`, `visibleSections` filter, sidebar
render logic, `FunnelStage`/`ParcoursSection` inline-expand rework,
9× `ExportBar` relocation, renumbering). `AROM-Documentation` (this
file).

## Test de fumée

- [x] Sidebar shows Approvisionnement/Production/Stock/Commercialisation
      indented under "01 Parcours production," always visible
- [x] Clicking any of them opens that exact same full page, tabs and
      all, unchanged
- [x] Inviting a "Personnalisé" staff member can still individually
      pick Approvisionnement (and the other three) in the section
      picker
- [x] Clicking "Voir le détail" on a Parcours stage expands an inline
      panel and rings the card, without leaving the page
- [x] "Ouvrir ... en entier" navigates to that section's full page
- [x] The Campagne/Export card renders at the bottom of every page
      that has one

Covered by the automated regression suite (`node regression-suite.mjs`,
51/51 passing, two consecutive clean runs).
