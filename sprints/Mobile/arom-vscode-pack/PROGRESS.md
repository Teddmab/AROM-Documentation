# AROM Mobile Completion Plan — Progress Tracker

Reconciled against actual repository state on 2026-09-16. This file lives
inside the pack (untracked in git, like the rest of `sprints/Mobile/` —
see "Working-tree state" below) and should be updated at the end of every
sprint, not just read at the start.

## Status legend

- ✅ **Complete** — verified against real code/tests/rules, not just a commit message.
- 🟡 **Partially complete** — real, shipped work exists; a specific listed gap remains.
- 🔁 **Superseded** — the original instruction was replaced by a later, documented decision.
- ⬜ **Not started**.
- 🚧 **Blocked** — a real business/product decision is required before more can be built.
- 🔍 **Remaining verification only** — implementation appears done; needs a native/device pass.

## Summary table

| # | Prompt | Status | Evidence |
|---|---|---|---|
| 01 | `01-documentation-and-flags.md` | ✅ Complete | AROM-Mobile `login.tsx`, `oauthConfig.ts`, `getPaymentGateway.ts`, `easConfig.test.ts`; AROM-Documentation `architecture.md`/`data-model.md`/`roadmap.md` content (dirty, uncommitted, but correct) |
| 02 | `02-freshness.md` | ✅ Complete | `refreshCoordinator.ts`, `useAutoRefresh.ts`, `SyncStatus.tsx` + 4 test files |
| 03 | `03-security.md` | 🟡 Partial | poste deny-by-default done; **missing**: read-only staff-poste audit report/script, and `isValidStockMP`/`isValidClients`/`isValidVentes`/`isValidMarketing`/`isValidCharges` in `AROM-Backend/firestore.rules` |
| 04 | `04-navigation-and-drafts.md` | ✅ Complete | `activeMode.tsx`, "Passer en mode ..." wording, reception→producer draft-return, payment-waiting safe-to-leave screen, `activeMode.test.tsx` |
| 05 | `05-reception-and-production.md` | ✅ Complete | Reception: `ProducerStep.tsx`, `receptionStatus.ts`. Quality journey: `productions.tsx`, `qualityRepository.ts` head-control walk, `resolve.tsx`. Production/packaging redesign: **AROM-Mobile `25996b9`, `4e1c4c9`** (this session) |
| 06 | `06-sales.md` | 🟡 Partial | Wizard order close but client is still mandatory, lot picker is manual/mandatory and shows lot IDs, no +/- controls, **no availability check of any kind exists** (not even the old estimate) |
| 07 | `07-admin.md` | 🟡 Partial | 4 distinct screens exist and wired correctly (`/activity`, `admin-summary`, `admin-priorities`, overview Home); honesty rules and stable priority keys verified; **gap**: `admin-operations.tsx` is still a full duplicate screen, not the "temporary redirect only" the prompt asks for |
| 08 | `08-finished-product-stock.md` | 🟡 Partial | **Steps A, B, C/D** (schema, QC-release receipts, order reservation/cancel/fulfil, dashboard integration) shipped and verified. **Blocker to landing**: the dashboard fix (`93ccda3`) sits on branch `fix/dashboard-order-reservation-repoint`, not yet fast-forwarded onto `AROM-Production` `main` — needs pre-existing unrelated dirty `dashboard.tsx`/`routeTree.gen.ts` changes reconciled first (repo hygiene, not a product decision). **Steps E/F** (direct-sale, UI) **not built** |
| 09 | `09-partner-orders.md` | 🚧 Blocked | Explicitly gated on business approvals (payment ownership, formats/prices, reservation duration/release policy, customer data, partner naming) — no approval record found anywhere in AROM-Documentation |
| 10 | `10-field-validation.md` | ⬜ Not started | Depends on 06/07/08/09 being substantially done first |

## Detail per sprint

### 01 — Documentation and flags — ✅ Complete
- `AROM-Documentation/architecture.md:127,138` documents AROM Mobile in the topology.
- `data-model.md:209-230` documents `producerInvoices`/`harvestOffers`/`harvestInvoices`, and records the decision that a future `partnerOrders` domain must never reuse `producerInvoices`/`mombongoInvoices`.
- `roadmap.md:280` marks the old combined-sprint Mombongo vision superseded, with corrected status.
- `AROM-Mobile/src/app/login.tsx:53-57` + `src/lib/firebase/oauthConfig.ts` hide Google/Facebook buttons when client IDs are unconfigured.
- `src/features/payments/getPaymentGateway.ts` + `eas.json` (`preview` only, never `production`) + `__tests__/easConfig.test.ts` guard the Mombongo simulation flag.
- Note: the six core AROM-Documentation files show as `git status` modified/untracked — pre-existing, not part of any commit yet. Content is correct regardless.

### 02 — Freshness — ✅ Complete
- `src/features/sync/refreshCoordinator.ts` — `refreshRolePlan()`, in-flight dedup map, `FOCUS_REFRESH_THROTTLE_MS = 60_000`, `success/partial/failed/offline/throttled/deduped` outcomes.
- `src/features/sync/useAutoRefresh.ts` — foreground (`AppState` "active") + focus-triggered (ADMIN) refresh, obsolete-result guarding via a key ref.
- `src/features/home/components/SyncStatus.tsx` — staleness threshold + last-sync display.
- Tests: `refreshCoordinator.test.ts`, `useAutoRefresh.test.tsx`, `SyncStatus.test.tsx`, `refreshFreshness.test.ts`.

### 03 — Security — 🟡 Partial
Done:
- `src/features/auth/roleAuthorization.ts`'s `getAuthorizedRoles()` denies by default for missing/unrecognized/"Personnalisé" poste.
- Webhook-only sensitive transitions preserved (`isValidMombongoWebhookTransition`, `isValidInvoiceTransition`, `isValidHarvestInvoiceTransition`).
- Strong emulator Rules test coverage for existing validated collections (`AROM-Backend/tests/rules.test.mjs`).

Not done:
- No read-only report (script or screen) listing active accounts with a missing/unsupported/"Personnalisé" poste.
- No migration-impact report or approval gate for existing unscoped accounts.
- `AROM-Backend/firestore.rules` has no `isValidStockMP`/`isValidClients`/`isValidVentes`/`isValidMarketing`/`isValidCharges` — these 5 collections are role-gated only, no field-shape validation (unlike `isValidProducteur`/`isValidProduction`/etc., which already exist).

This remainder is small, self-contained, and does not conflict with sprint 08 — a reasonable follow-up sprint.

### 04 — Navigation and drafts — ✅ Complete
- `src/features/home/activeMode.tsx` (`ActiveModeProvider`) is the sole owner of `activeMode`, re-validated against `getAuthorizedRoles` on every switch.
- `"Passer en mode ..."` wording confirmed in `more.tsx:107` and `AdminReadOnlyGate.tsx`.
- Reception→producer draft-return: `reception-new.tsx` passes `returnTo`/`receptionDraftId`; `producteur-new.tsx` reads it, runs a "minimal mode," and returns via `onUseForReception`. Tested in `ProducteurWizardScreen.test.tsx`.
- Payment-waiting safe-to-leave: `producer-invoice/[id]/pay.tsx`.
- Per-wizard exit-protection tests exist across `ReceptionWizardScreen.test.tsx`, `ProductionWizardScreen.test.tsx`, `SaleWizardScreen.test.tsx`, plus `activeMode.test.tsx` for mode-switch/unauthorized-refusal.

### 05 — Reception and production journeys — ✅ Complete
**Reception**: `ProducerStep.tsx` ("Qui livre les fruits ?", recent producers, local search, add-new), minimal producer creation (`IdentityStep`/`ContactStep`/`ActivityStep`, skips photo mid-reception), auto-return to the same draft, `VerifyStep.tsx` plain-language recap, `receptionStatus.ts`'s explicit Brouillon/En attente/Synchronisation/Synchronisé/Échec state machine.

**Quality/production journey**: `productions.tsx` derives À terminer / Contrôle qualité à faire / Productions récentes sections from sync history + quality lots, one action per row. `StatusBadge.tsx` labels match spec exactly. `qualityRepository.ts` walks the `resolvesId` chain to find the latest *effective* control (not latest-by-timestamp). `quality-control/[id]/resolve.tsx` creates a linked resolution control rather than mutating the original.

**Production quantity/output/packaging redesign** (a deeper elaboration of this same sprint, per `manual-entry-automation-audit.md`): done and committed this session.
- `4e1c4c9` — visual quantity feedback (fruit-crate/juice-container/sample-container/bottle-count `QuantityVisual`, reception comparison).
- `25996b9` — Quantité utilisée / Résultat obtenu / Embouteillage step redesigns, shared `MeasurementBar`/`QuantityStepperInput`, format-change confirmation.
- Do not reopen without a reproduced regression (per this session's own instruction).

### 06 — Product-first sales — 🟡 Partial (real gap, not superseded)
- `saleDraft.ts`'s `WIZARD_STEPS = ["details", "lots", "client", "payment", "verify"]` — order is close, but:
  - `validateClientStep` still requires `idClient` — client is **mandatory**, not optional as specified.
  - `LotsStep.tsx` is a manual, **mandatory** multiselect showing `production.lot` (the lot ID) directly during normal entry — the opposite of "do not show production-lot IDs during normal sale entry" and "allocate eligible production lots automatically... ask the user only if automatic allocation cannot succeed."
  - `DetailsStep.tsx` uses plain numeric `TextField`s — no large minus/plus controls anywhere in `src/features/sale/components/*`.
  - **No finished-product availability check exists anywhere in the sale flow** — not the old estimate, not the new `stockBalance` model. Nothing to swap; this needs to be built.
- Only 2 commits ever touched this area (`eb9f4a0` MOB-03–11, `99d1129` post-MOB-11 pass) — the sales-specific redesign was never actually done in either.
- **Real dependency confirmed**: the sprint's own text says not to estimate availability and to record the dependency on sprint 08 — automatic FIFO allocation belongs naturally on top of `stockBalance`/`stockLotBalance` (sprint 08 Step E, "direct sales"), so finishing this properly is best sequenced *after* sprint 08's Steps C–E, not before.

### 07 — Administrator experience — 🟡 Partial
- Four distinct, correctly-wired screens: Home (`AdminOverviewScreen.tsx`), Activity (`/activity`), Operational Summary (`admin-summary.tsx`), Priorities (`admin-priorities.tsx` + detail).
- Home's card already reads "Voir l'activité récente" → `/activity`; zero remaining navigation entry points to `/admin-operations` (confirmed by grep).
- Honesty rules verified: `adminOperations.ts` never infers health from a zero count; `admin-summary.tsx` shows a stale-data banner; `adminSummary.ts` tallies the already-deduped head-control result (no double-counting quarantine + resolution).
- `adminPriorities.ts` uses stable composite keys (e.g. `` `quality-awaiting-${productionId}` ``).
- **Gap**: `src/app/admin-operations.tsx` is still a full 154-line duplicate screen with its own data fetching, not converted to the "temporary redirect only" the prompt asks for.

### 08 — Finished-product stock and reservations — 🟡 Partial
**Step A (schema) — done.** `AROM-Mobile` `4f757ad` (`StockBalance`/`StockLotBalance` discriminated union, `allocateFifo`, canonical-format mapping), mirrored in `AROM-Backend` `25a2770` (Rules shapes + migration/reconciliation dry-run script) and `AROM-Production` `58b6994` (canonical formats) + `b004818` (QC-gated finished-stock projection fix).

**Step B (QC-release receipts) — done.** `AROM-Mobile` `e3ea653`/`01cb75e`/`e5f43a3` (stockPF write → routed through the trusted inventory service, actor-identity/lost-ack hardening), `AROM-Backend` `fc096fb`/`ea218bc` (Rules restrict `stockPF`/`stockBalance`/`stockLotBalance` writes to the `isInventoryService()` claim only, provisioning script, 239-test Rules suite), `AROM-Production` `7cf6379` (`/api/inventory/qc-release` trusted route + `qcReleaseReceipt.ts` transaction, 556-line test file).

**Steps C/D (order confirm/cancel/fulfil) — done. Dashboard integration gap now closed.**
- `AROM-Production` `17d2277`: `src/lib/inventory/stockBalance.ts` (ported FIFO model), `src/lib/inventory/orderReservation.ts` (`confirmOrderReservation`/`cancelOrderReservation`/`fulfilOrderReservation` — transactional, idempotent, validates every item against a live `products/{id}` doc, allocates FIFO across `stockLotBalance`, writes the `stockPF`/`ventes` bridge rows on fulfil), `src/lib/auth/verifyCommercialInventoryCaller.ts` (admin or "Chargée de Commercialisation"), 3 new trusted routes `/api/inventory/{confirm-order,cancel-order,fulfil-order}.ts`. 93/93 vitest passing, `tsc --noEmit` clean, `vite build` clean (route tree registers all 3).
- `AROM-Backend` `d7385bf`: `firestore.rules` tightens `orders/{id}.update` to a 3-branch rule — `isInventoryService()` + a validated `isValidOrderStatusTransition` for the confirm/cancel/fulfil status+reservation change; ordinary staff keep free edit of every other field; partner pending-order self-cancel unchanged. 10 new Rules tests, suite 283/283.
- `AROM-Backend` `684702e` (**follow-up fix**): `d7385bf`'s ordinary-staff branch required `status` fully unchanged, which unintentionally also blocked staff from cancelling a still-*pending* order (a transition that never touches stock/reservation, and was already a documented-safe carve-out for partners). Extended the staff branch to also allow pending→cancelled, mirroring the partner carve-out. 4 new tests, suite 287/287.
- `AROM-Mobile` `38ef0db`: `orderReservationClient.ts` (new, mirrors `inventoryReleaseClient.ts`), `orderActions.ts` rewritten (`confirmOrder`/`cancelOrder`/`fulfillOrder` call the trusted routes; `cancelPendingOrder` keeps the original direct write for the still-safe pending-order case), `orders.tsx` repointed (confirmed via `OrderDetailModal`'s own JSX that the cancel button is structurally unreachable for non-pending orders, so it calls `cancelPendingOrder` only). Full Jest suite 1525/1525, `tsc`/`eslint` clean, `expo export --platform web` clean.
- `AROM-Production` `93ccda3` (**dashboard integration gap, closed**): `dashboard.tsx`'s "Commandes boutique partenaires" card, the duplicate task-driven `fulfillOrder` helper, and the order-confirm task-completion handler (5 call sites total) repointed to `confirmOrderTrusted`/`cancelOrderTrusted`/`fulfilOrderTrusted`. The still-pending-order cancel button is unchanged (direct write, now confirmed Rules-safe by `684702e` above). Built on a **clean git worktree off `17d2277`** rather than the dirty working tree, since `dashboard.tsx`/`routeTree.gen.ts` there carry large, unrelated, pre-existing uncommitted changes (task-stage-scoping security hardening, reception photo-evidence display, and an uncommitted Mombongo-integration slice) that predate this fix. Added a focused static-protection test (`dashboard.protected-orders.test.ts` — no React component-test infra exists in this repo) asserting no direct write of `status: "confirmed"`/`"fulfilled"` or a `reservation` field remains. 98/98 vitest, `tsc`/`eslint`/`vite build` all clean.
  - **Update (2026-09-17, later same day): landed on `main`.** A dedicated stabilization session first versioned the previously-uncommitted Mombongo-integration slice as `AROM-Production` `2db292a` (`feat(mombongo): version existing invoice integration` — outbound calls, inbound webhook, 5 mobile-facing routes, 152 tests; see `AROM-Documentation/mombongo-integration-audit.md` and `architecture.md`'s "Mombongo integration — status" section), then cherry-picked `93ccda3` on top as `a33bda1`. The other unrelated uncommitted work (`tasks.ts`, `auth.tsx`, `login.tsx`, `join.tsx`, `storefront/signup.tsx`, and dashboard's task-stage-scoping/photo-evidence changes) was preserved, not lost — see branch `recovery/pre-dashboard-repair-dirty-work` on `AROM-Production`. **Sprint 08 Steps C/D, including the dashboard integration gap, are now fully done and on `main`.**

**Steps E/F — not built.** Fully specified, pre-approved architecture exists at `AROM-Documentation/automation-engine.md` lines 217-476 ("Reservation and balance projections — approved architecture (Sprint 08, 2026-09; not yet implemented)") and its own stated build order at line 664: *"Step A ... before B ... before C/D (order reservation/fulfilment, meaningless without B already trusted) before E (direct sales) before F (UI)."*
- **E — direct sales**: repoint `AROM-Production`'s `CommercialisationSection` and `AROM-Mobile`'s `sale-new` at a `direct-sale` trusted endpoint.
- **F — UI**: sprint 06's own remaining sales-wizard work, naturally sequenced after E.

→ **Next up: mobile Mombongo invoice reflection (backend baseline now ready — see below), then Step E (direct sales).**

## Mombongo invoice reflection — 🟡 In progress (2026-09-17, later same day)

Backend baseline landed (`AROM-Production` `2db292a` + `a33bda1`, above). **Mobile side remains incomplete**: `AROM-Mobile`'s `producer-invoices.tsx` ("Factures") has no auto-refresh of its own — a new invoice (whether `producerInvoices` or Mombongo-originated `harvestInvoices`) only enters the offline cache when `refreshRolePlan` re-runs via `useAutoRefresh`, which is wired up only on `AdminOverviewScreen`. `harvest-invoice/[id]/pay.tsx` also claims automatic status updates it doesn't deliver (no `onSnapshot`). Full detail in `AROM-Documentation/mombongo-integration-audit.md` §1.2. This is the scope of a follow-up mobile-focused session — not done as part of the `AROM-Production` stabilization above.

**External Mombongo API remains unavailable** (`createExternalInvoice`/`getExternalPublishedListings` both HTTP 500 on their dev host, reconfirmed 2026-09-17 — a regression on their side since 2026-08-31, outside AROM's control).

### 09 — Partner-order pilot — 🚧 Blocked
No approval record found anywhere in AROM-Documentation for any of the 6 explicit gate items (payment/refund ownership, official formats/prices, reservation duration/release policy, minimum customer/delivery data, canonical partner naming). Do not start. Unaffected by, and not advanced by, the Mombongo invoice-reflection work above — that work is limited to `harvestInvoices`, not `partnerOrders` or any customer-order flow.

### 10 — Field validation and release — ⬜ Not started
Depends on 06/07/08/09.

## Next sprint

**Sprint 08, Steps C/D — order reservation, cancellation, and fulfilment: done and committed** (`AROM-Production` `17d2277`, `AROM-Backend` `d7385bf`, `AROM-Mobile` `38ef0db`). One repointing gap remains — see the Step C/D gap note above — before Step E can start cleanly, since Step E would add a 4th trusted route to the same `dashboard.tsx` file.

**Selected next: finish the `dashboard.tsx` repointing gap, then Sprint 08 Step E (direct sales).**

Owning file: `prompts/08-finished-product-stock.md`, same approved architecture in `AROM-Documentation/automation-engine.md`.

Not sprint 03's remainder (earlier in pack order, but smaller/independent — flagged above as a good follow-up, does not block or get blocked by sprint 08).
Not sprint 06 (depends on Step E's trusted direct-sale route to do automatic allocation honestly — attempting it first would mean re-deriving the same trusted-route pattern twice).
Not sprint 09 (explicitly gated, unrelated collection — the existing `orders` collection this sprint touches is the pre-existing storefront/partner-order path, not the future gated `partnerOrders` domain).

See the checkpoint report and this session's transcript for exact scope, dependencies verified, stop conditions, and affected repositories.

## Working-tree state (updated after Step C/D commits, 2026-09-16)

- **AROM-Mobile**: clean (Step C/D work committed at `38ef0db`).
- **AROM-Backend**: clean (Step C/D work committed at `d7385bf`).
- **AROM-Production**: Step C/D's own new files committed at `17d2277`. Still dirty, left untouched: `src/lib/erp/tasks.ts`, `src/lib/firebase/auth.tsx`, `src/routes/join.tsx`, `src/routes/login.tsx`, `src/routes/storefront/signup.tsx` (pre-existing, unrelated, not mine), `src/routes/dashboard.tsx` and `src/routeTree.gen.ts` (pre-existing unrelated changes **plus** this session's still-uncommitted repointing edit — see the Step C/D gap note above), `.playwright-mcp/` and ~90 scratch screenshots (pre-existing), and a pre-existing uncommitted Mombongo-integration slice (`src/lib/auth/verifyMombongoCaller.ts`, `src/lib/firebase/auth-errors.ts`, `src/lib/payments/*.ts`, `src/routes/api/mombongo/`, `src/routes/api/webhooks/`) that predates this session and was never touched by it.
- **AROM-Documentation**: `architecture.md`, `data-model.md`, `flows.md`, `rbac.md`, `roadmap.md`, `runbook.md` modified; `sprints/Mobile/` untracked (the whole completion-plan folder, including this tracker). Pre-existing from earlier sessions — left untouched, not staged, not committed as part of this sprint.
