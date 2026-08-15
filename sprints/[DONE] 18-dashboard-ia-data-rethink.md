# Sprint 18 — Dashboard information architecture, data/logic fixes, KYC review, password reset

**Status:** Done — [Teddmab/AROM-Production#14](https://github.com/Teddmab/AROM-Production/pull/14)

**Rôle concerné :** Admin / Staff (toutes les pages du dashboard)
**Page / zone :** Dashboard — Primes & personnel, Commercialisation, Stock, Finances, Paramètres ERP, Exécutif ; `/login`

## Pourquoi maintenant

Requested directly after a page-by-page audit of the dashboard (done in
an earlier session) surfaced several things that had never been
actioned: the Campagne/Du/Au filter bar present on every section only
ever fed PDF/Excel export, never what's rendered on screen; `recharts`
was a dependency but no chart existed anywhere; `charges` (fixed costs)
had no admin UI at all; campaign dates in Paramètres ERP were
read-only. Separately, the client flagged directly: "Primes et
Personnel [...] the different cards should be a sub-menu instead of
having everything there, same for different other pages" and "we are
still missing a way to even approve the KYC" — the latter traced to
`BoutiquesCard`'s "Vérifié" toggle showing nothing to actually review
(the `idNumber` collected at signup, sprint 13, was never fetched into
that card). A fresh audit at the same time found no password-reset
flow anywhere in the app.

Explicit requirement from the client: not another narrowly-tailored
change (see [sprints/17](<[DONE] 17-staff-poste-data-enforcement.md>))
— a rethink across pages, data, and logic, not one feature.

## Dans le périmètre

- **Tabbed sub-navigation** for the two sections that had stacked 5-6
  cards into one long scroll: Primes & personnel (Invitations / Équipe
  / Boutiques partenaires / Primes production / Primes commercial) and
  Commercialisation (Catalogue / Promotions / Commandes / Ventes &
  clients). Every other section stayed as-is — they're already 1-3
  cards, and forcing tabs onto a small page adds a click for no
  benefit.
- **ExportBar filter now actually filters on-screen data.** Lifted the
  Campagne/Du/Au filter out of `ExportBar`'s local state into the
  shared `ErpContext`, so `computed` is derived from `filterErpState(state, filter)`
  everywhere, not just inside `buildFilteredReport()`. One shared
  filter across the whole dashboard (not per-section) — "pick a
  campaign, browse every page through that lens" is the actual mental
  model.
- **Stock cumulé fixed to chronological order.** `stockMP`'s running
  balance is order-sensitive; the collection now carries a query-level
  `orderBy("date")` (the one collection whose consumer does
  order-sensitive math) instead of relying on Firestore's default
  (insertion/doc-id) order.
- **Charges fixes admin card.** `charges` was read into KPIs but had
  no create/edit/delete UI anywhere — now a card in Finances,
  following the same `EntryForm`/`Table`/`addRow`/`removeRow` pattern
  every other collection already uses.
- **Campaign dates editable.** Paramètres ERP's "Calendrier de
  campagne" card was a read-only table; now three date inputs wired to
  `updateParametres`, same pattern as the existing numeric fields.
- **A real chart.** Exécutif — the page every role lands on first —
  now shows a production-volume / ventes-revenue trend chart (recharts,
  via the already-built but previously-unused `ChartContainer`
  wrapper). Not rolled out to every section in this sprint (see
  Backlog below).
- **KYC review.** `BoutiquesCard` now shows the CNI/RCCM number
  collected at signup (`idNumber`) next to the "Vérifié" toggle — admin
  previously had nothing to look at before verifying. Added an
  unverified-first sort so new signups aren't buried in an alphabetical
  list.
- **Password reset**, missing entirely before this sprint — "Mot de
  passe oublié ?" on `/login`, backed by Firebase Auth's own
  `sendPasswordResetEmail`.

## Hors périmètre

- Whether Paramètres ERP / Feuille de route belong under a settings
  submenu — raised in the earlier audit, no clear win either way, and
  a much smaller problem than the two sections that got tabs.
- A document-upload + admin-review KYC workflow. `idNumber` is still
  free text collected at signup (sprint 13's explicit scope decision,
  unchanged); this sprint only makes the text actually visible to
  admin. `verified` remains informational, never a checkout gate (see
  [rbac.md](../rbac.md#poste-based-data-scoping-sprint-17), unchanged
  from sprint 16).
- Charts beyond the one Exécutif trend chart; table pagination/search;
  storefront inventory/stock-out prevention; full admin/staff account
  management UI (deactivate, change role); new-signup notifications;
  email verification. All real gaps, all bigger scope than this
  sprint — captured in [roadmap.md](../roadmap.md#backlog-toward-viable-v1)
  as the answer to "list everything that needs upgrading for a viable
  V1," not silently dropped.

## Décisions déjà actées

- **One shared filter, not per-section.** Simpler than per-section
  filter state, and matches how a campaign/date lens is actually used
  — picked once, applied everywhere.
- **Tabs only where the card count actually justified it.** A fresh
  structural map of `dashboard.tsx` found exactly two outlier sections
  (Personnel, Commercialisation); every other section is 1-3 cards
  already. Reused the existing `src/components/ui/tabs.tsx` (a full
  shadcn/Radix wrapper that had never been imported anywhere) instead
  of building new sub-nav.
- **`verified` stays informational.** Not reopening sprint 16's
  decision that boutique verification never gates checkout — this
  sprint closes the "nothing to review" gap, not the "should this
  block orders" question, which wasn't what was reported broken.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` or
`AROM-Backend` change needed for anything in this sprint.

## Livrable

`AROM-Production`: `src/routes/dashboard.tsx` (tabs, Charges card,
editable campaign dates, trend chart, BoutiquesCard KYC columns),
`src/lib/erp/store.tsx` + `src/components/erp/ExportBar.tsx` (shared
filter, stockMP ordering), `src/lib/firebase/auth.tsx` +
`src/routes/login.tsx` (password reset). `AROM-Documentation` (this
file, data-model.md, roadmap.md).

## Test de fumée

- [x] Primes & personnel and Commercialisation render as tabs (5 and 4
      respectively), each tab showing what used to be a stacked card
- [x] Setting the Campagne/Du/Au filter on any section changes the
      on-screen numbers, not just what a PDF/Excel export would contain
- [x] A `stockMP` entry logged with an earlier date than existing rows
      sorts to the top and its "Stock cumulé" reflects chronological
      order, not insertion order
- [x] Admin can add and delete a fixed charge from the new Finances
      card
- [x] Campaign dates in Paramètres ERP are editable and persist
- [x] Exécutif shows a real production/ventes trend chart
- [x] Boutiques partenaires shows the CNI/RCCM number collected at
      signup, sorted unverified-first
- [x] "Mot de passe oublié ?" on `/login` sends a reset and shows a
      confirmation

Covered by the automated regression suite (`node regression-suite.mjs`,
36/36 passing, two consecutive clean runs).
