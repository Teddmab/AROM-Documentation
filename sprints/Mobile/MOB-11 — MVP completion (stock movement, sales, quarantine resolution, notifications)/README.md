# MOB-11 — MVP completion: stock movement, sales, quarantine resolution, notifications

> **Status: Done** (2026-09-01). Written after the fact, same as Sprint DP —
> this pass closed the last dishonest surfaces in the app (two
> `kind: "comingSoon"` role-home stubs and one long-deferred quarantine
> gap) so the mobile app has no dead ends left for a presentable MVP.

## Why this sprint exists

A full audit of MOB-01 through MOB-10 (see each sprint's own updated
status banner) found the app largely complete and compliant with its
specs, but surfaced three real gaps that were never speced anywhere:

1. PRODUCTION role's home ("Production **& stock**") had a "Mouvement de
   stock" action routed to the generic `/coming-soon` stub — no
   stock-movement entry flow had ever been built, despite the role's own
   name promising it.
2. SALES role's home ("Ventes & boutique") had the same stub on
   "Nouvelle vente" — no sale-entry flow existed at all; `orders.tsx`
   only ever handled storefront order fulfillment, never a manually
   logged direct sale.
3. MOB-05B's own "Confirmed decisions" explicitly deferred quarantine
   *resolution* ("MOB-05C") — a quarantined production lot had no way to
   ever be re-decided, and `adminPriorities.ts` showed it with literal
   "fonctionnalité à venir" copy.

A fourth item — a real notification center — was added to this same
pass at the user's explicit request when scoping the work (the header
bell was a static, non-interactive placeholder before this).

## What shipped

### Stock movement (`src/features/stockMovement/`, PRODUCTION role)

A full offline-capable entry flow for `stockMP` — mirrors the
`MouvementStockMP` shape from `AROM-Production/src/lib/erp/model.ts`
exactly (type Entrée/Sortie/Ajustement, linked réceptions for an Entrée,
linked production lots for a Sortie). Sync mechanics copy
`receptionSync.ts` (plain idempotent `setDoc`) rather than
`productionSync.ts`'s transaction/conflict machinery — deliberately, since
a stock movement's source/destination lots have no "already consumed"
concept the way a single-use reception does. New routes:
`/stock-movement-new`, `/stock-movements`, `/stock-movement/[id]`.
`adminOperations.ts`'s "Stock matières premières" card now points here
instead of `/coming-soon`.

### New sale (`src/features/sale/`, SALES role)

Same treatment for `ventes` — mirrors `Vente`/`Client`/`Canal`/`Format`
from the web model exactly, including the real, previously-unenforced-on-
mobile requirement that a sale reference at least one source production
lot ("Lots sources", matching the web dashboard's own form). SALES's
sync plan (`syncPlan.ts`) gained a `productions` category to support this
picker. New routes: `/sale-new`, `/sales`, `/sale/[id]`.

### Quarantine resolution — MOB-05C (`src/features/quality/`, PRODUCTION role)

The highest-risk piece: `qualitySync.ts`'s existing "one control per lot"
transactional invariant had to be *relaxed*, not removed, to allow
*exactly one* second document — a resolution — per quarantined lot,
while a completely fresh (non-resolving) second control on an
already-controlled lot is still rejected exactly as before. A resolution
is always a brand-new `qualityControls` document (`resolvesId` pointing
at the quarantine it resolves), never an in-place edit of the original —
preserving the "durable audit record, never overwritten" principle
`model.ts` already documented for this collection. `qualityRepository.ts`
now derives a lot's *current* control as the "head" of a resolution
chain (the control nobody has resolved yet) rather than assuming exactly
one control ever exists per lot. New route:
`/quality-control/[id]/resolve`. `adminPriorities.ts`'s quarantine item
now points there instead of showing "fonctionnalité à venir".

### Notification center (`src/features/notifications/`)

The header bell (`AppHeader.tsx`) is now a real, tappable action instead
of a static icon. ADMIN's tap goes to the existing `/admin-priorities`
(already comprehensive for that role); COLLECTION_AGENT/PRODUCTION/SALES
get a new `/notifications` screen, reusing `adminPriorities.ts`'s own
`PriorityItem` shape and `PriorityRow` component. Content is built only
from data already computed elsewhere — pending tasks (same `instructions`
sync category every role already caches), this device's own failed
sync attempts (`mutationQueue.ts`, scoped per role to the entities that
role actually writes), plus role-specific real signals: quarantined lots
awaiting resolution for PRODUCTION, orders awaiting confirmation/
fulfillment for SALES. No invented thresholds, no unread-count badge
(a live count would mean fetching notification data on every screen
mount app-wide — the icon stays a plain, honest affordance instead).

## Verified

`bun run typecheck`, `bun run lint`, and `bun run test` all clean —
952 tests passing (up from 732 before this pass), including new coverage
for the quarantine-resolution transactional edge cases (fresh-control
regression guard, valid resolution, duplicate-resolution rejection,
resolution-of-a-non-quarantine rejection) and every new screen/repository/
sync file. `grep -rn "coming-soon" src/features/home/roleHomeConfig.ts
src/features/admin/adminOperations.ts` returns nothing — no
`kind: "comingSoon"` role-home action remains anywhere in the app.

No live device/Expo Go pass was performed (same limitation prior sprints
— e.g. MOB-06B — already flag honestly for their own work); verification
here is code/test-level only.
