# MOB-06B — implementation notes (as built, 2026-08-30; Workstream A added 2026-08-31)

## Scope actually delivered this pass

The sprint has two workstreams. This pass delivered **Workstream B in
full** (mock payment gateway + guided B1–B7 payment journey) plus the
**A7 slice of Workstream A** (the invoice list and verify/reconcile
screens), because the user supplied the payment-flow visual reference for
exactly these screens. **A1–A6 (Vue d'ensemble, Activité, Opérations,
Priorités, Équipe, Détail priorité) are not redesigned yet** — no visual
reference for those has been supplied; building them without one risks
visible rework once it arrives. The shared guided components built here
(`GuidedPageIntro`, `WhatNextCard`, `IllustratedStatus`,
`GuidedPrimaryAction`, `DemoDataBanner`) are reusable as-is for that
follow-up pass.

## Mock payment architecture

New `src/features/payments/`:

- `gateway.ts` — the `ProducerPaymentGateway` interface
  (`getCapability()`/`startPayment()`), `PaymentCapability`,
  `PaymentResult`, `DemoPaymentOutcome` types.
- `unavailableGateway.ts` — the only gateway a production build can ever
  resolve to. Replaces MOB-06A's `producerPaymentCapability.ts` (deleted).
- `getPaymentGateway.ts` — the single DI seam. Requires **both**
  `__DEV__` (RN's built-in global, `false` in every release build,
  un-overridable by any env var) **and** the explicit
  `EXPO_PUBLIC_ENABLE_MOCK_PAYMENTS === "true"` flag (same
  `process.env.EXPO_PUBLIC_*` convention `lib/firebase/config.ts` already
  uses for the Firestore emulator). No screen or hook checks either gate
  itself — everything goes through this one function.
- `mock/mockProducerPaymentGateway.ts` — the demo simulation. Reached only
  via a lazy `require()` inside the `__DEV__`-gated branch of
  `getPaymentGateway.ts`, never imported unconditionally. Self-contained:
  no `firebase/firestore` import, no call into `invoices/invoiceSync.ts`,
  no read/write of the real `producerInvoices` collection or cache —
  verified by a test that statically inspects its own import statements,
  not just its runtime behavior. Idempotent by `idempotencyKey`:
  `startPayment` is deliberately **not** an `async function` (which would
  wrap every call in a fresh Promise) so a duplicate key returns the
  literal same in-flight-or-resolved Promise object, not just an equal
  value.
- `mock/mockScenarioController.ts` — the only way to reach
  `pending`/`failure`/`timeout` (default is `success`, for a predictable
  demo happy path). No production-facing UI control exists yet; tests
  inject scenarios directly.

## Screens

- `app/producer-invoice/[id].tsx` → moved to `app/producer-invoice/[id]/
  index.tsx` — **required**, not cosmetic: Expo Router doesn't support a
  leaf file and a directory both claiming the same dynamic segment
  (`[id].tsx` alongside `[id]/pay.tsx`). Redesigned: `GuidedPageIntro`,
  `InvoiceComparisonCard` (replaces three separate reconciliation
  `StatusBanner`s), `WhatNextCard`, and — only when
  `getPaymentGateway().getCapability()` reports `mode: "demo"` — an
  enabled `GuidedPrimaryAction` "Payer la facture" that opens the wizard.
  In every other mode (`"unavailable"`, i.e. production or dev without the
  flag), behavior is byte-for-byte what MOB-06A shipped: disabled button,
  reason banner, no route to the wizard exists.
- `app/producer-invoice/[id]/pay.tsx` — new wizard, one route, local
  `useState<"verify"|"confirm"|"processing"|"result">`, mirroring
  `quality-control-new.tsx`'s wizard-in-one-screen pattern. Confirm step
  is a `Modal` bottom sheet over the verify screen (same structure as
  `QuickActionSheet.tsx`). Processing is non-dismissible: `AppShell`'s
  `confirmNavigation` swallows every BottomNav/quick-action attempt, the
  header back button is hidden (`onBack={undefined}`), and
  `BackHandler.addEventListener("hardwareBackPress", () => true)`
  swallows the Android back button, all only while `step === "processing"`.
  Guards defensively redirect back to the invoice detail if the capability
  isn't actually `"demo"` at render time — this route must never be
  reachable in production even by a stale deep link.
- `app/producer-invoices.tsx` — `GuidedPageIntro` + real summary chips (À
  vérifier / À payer / Payées, counted from real `sync.invoices`, never
  fabricated) + a `WhatNextCard` when review is needed.

**Demo banner scoping**: shown starting at "Facture prête à payer" (once
`mode === "demo"`) through every wizard screen — never on the plain
invoice list or the review/reconcile screens, which are real business
logic regardless of payment mode. The reference image shows the banner on
every board screen, but the README's own "written requirements override
illustrative values" clause and the acceptance criterion's narrower
"every *payment* screen" wording support this scoping — see the sprint
plan for the full reasoning.

**Illustrations**: every photographic illustration slot in the reference
(hero images, checkmark/hourglass/plant graphics) is implemented as a
large icon badge (`IllustratedStatus`) using the app's existing 3-color
semantic system (`colors.success`/`warning`/`destructive` — the same
tints `StatusBadge`/`DecisionBadge`/`PriorityRow` already use), not a new
purple/lavender hue introduced just to match one reference image's
illustration color. Real photography for these screens hasn't been
supplied; swap in real assets when available.

## Superseded from MOB-06A

`payments/producerPaymentCapability.ts` and its test, and
`payments/components/PaymentFlowStates.tsx` and its test, are deleted —
the gateway architecture replaces the former, and the real wizard's result
screens replace the latter's presentational-only fixtures. Nothing else
referenced either file.

## Test results (2026-08-30)

- AROM-Mobile: **84 test suites / 702 tests, all passing**
  (`bun run test`), including full MOB-01–MOB-06A regression.
  `bun run typecheck` and `bun run lint` both clean.
  `npx expo export --platform web` succeeded with
  `EXPO_PUBLIC_ENABLE_MOCK_PAYMENTS` unset (a "production-shaped" build),
  producing all 31 routes including the two new/moved ones.
- AROM-Backend rules suite: unchanged this sprint (no rules edits), run
  twice via `firebase emulators:exec --only firestore,storage "npm test"`
  regardless, as a regression check. **Both runs: 74/77, identical 3
  failures both times** — the same pre-existing Storage-emulator
  cold-start race documented since MOB-06A, not a new failure.
- Manual grep checks: `payments/mock/`'s import statements contain no
  `firebase/firestore` reference; no call anywhere in `payments/` or
  `app/producer-invoice/` passes `"payee"`/`"paiement_en_attente"` to
  anything (outside `invoices/model.ts`'s own enum definition).

## Visual verification

No simulator, Expo Go instance, or screenshot tool was available in this
environment. Conformance to the reference was verified at the code level
(component order, exact required copy, state coverage — checkbox gating,
non-dismissible processing, all four result states, demo banner
placement) and by the full Jest/RTL suite above, **not** by device
screenshot comparison. Per the sprint's own instruction, this is reported
honestly rather than claimed as visual sign-off — a manual Expo Go/
emulator pass is still needed before calling Workstream B's visual
conformance complete.

## Workstream A — A1–A6 redesign (2026-08-31 pass)

Delivered once the user supplied `MOB-06B-guided-admin-redesign.png`
(the 6-panel Vue d'ensemble → Activité → Opérations → Priorités →
Équipe → Détail priorité board). Presentation-only: every screen keeps
its existing real data source (`useAdminDashboardData`,
`buildOperationsSummaries`, `buildActivityFeed`, `buildTeamActivity`,
`useAdminPriorities`) — no business logic or collection changed.

- New shared components: `guided/components/ModuleIconBadge.tsx` (inline
  icon-in-circle, reusing the app's existing per-role icons — the
  `COLLECTION_AGENT` truck, `PRODUCTION` cube, `SALES` storefront — rather
  than inventing new ones) and `guided/components/InitialsAvatar.tsx`
  (same photo-or-initial fallback as `ProducerStep.tsx`'s
  `ProducteurAvatar`; `equipe` members have no `photoUrl` field, so
  initials-only).
- `PriorityItem` (`adminPriorities.ts`) gained an optional `details`
  array — fact rows for A6, populated only from fields already in scope
  (lot/date/volume for quality items, réception/montant for invoice
  items). `adminActivity.ts`'s `ActivityItem` gained `kind` (for icon
  selection — `filter` alone conflates production and quality) and
  `needsAttention`. `adminOperations.ts`'s `ModuleSummary` gained
  `needsAttention`, true only for the qualité module and only when
  `useAdminPriorities` already surfaced a rejeter/quarantaine/awaiting
  item — no new threshold invented for any module.
- A1 (`AdminOverviewScreen.tsx`): hero image reuses
  `ROLE_CONFIG.ADMIN.hero` (already-real photography, already used on
  role-select — no new asset needed); the up-to-3-item priority list was
  replaced with one dominant callout (`priorities.items[0]`, or a calm
  "Tout va bien" empty state); the metrics grid was trimmed from 5
  modules to the reference's 3 (Réceptions/Production/Ventes — Qualité
  and Stock stay reachable from A3/A4); the separate "Factures
  producteurs" card was dropped (not in the reference; invoices needing
  review already surface as priority items).
- A2 (`activity.tsx`), A3 (`admin-operations.tsx`), A4
  (`admin-priorities.tsx` + upgraded `PriorityRow.tsx`), A5 (`team.tsx`):
  each gained `GuidedPageIntro` + restyled rows/cards using the new icon
  badges and status pills, plus a `WhatNextCard` guidance card, per the
  shared guided-screen contract. A4's `PriorityRow` keeps the 3-color
  semantic system (red/orange/muted) rather than the reference's
  illustrative purple — same precedent already set by
  `IllustratedStatus.tsx`'s own doc comment for the payment flow.
- A6 (`admin-priorities/[id].tsx`): rebuilt on `IllustratedStatus` +
  the new `details` fact-row card + `WhatNextCard` + `GuidedPrimaryAction`.
  This is also where the session's real bug lived and was fixed here:
  the CTA pushed `item.route` **without** `role: "ADMIN"` —
  `useHomeGuard` requires that param on every destination screen, so a
  missing one silently redirected the admin back through
  `/role-select` → home instead of opening the target screen. Now
  fixed and covered by a regression test
  (`AdminPriorityDetailScreen.test.tsx`, "pushes the CTA route with
  role: ADMIN").

**Test results (2026-08-31):** `bun run typecheck` and `bun run lint`
both clean. `bun run test`: **84 suites / 704 tests, all passing**
(702 prior + 2 new — A6's fact-row and `role: "ADMIN"` regression
tests). Existing suites for every touched screen
(`HomeScreen`, `ActivityScreen`, `AdminOperationsScreen`,
`AdminPrioritiesScreen`, `AdminPriorityDetailScreen`, `TeamScreen`,
plus the unit suites for `adminActivity`/`adminOperations`/
`adminPriorities`/`adminTeamActivity`) were updated only where the
redesign genuinely changed asserted copy (e.g. A1's old "À traiter
aujourd'hui" section title and "Factures producteurs" card no longer
exist) — every other existing assertion (exact status wording,
producer/priority titles, "Inactif", "Non disponible", etc.) was left
untouched and still passes against the new markup.

**Visual verification:** same limitation as the 2026-08-30 pass — no
simulator or Expo Go instance was available in this environment.
Conformance to `MOB-06B-guided-admin-redesign.png` was verified at the
code/test level (component order, exact required copy, the shared
guided-screen contract's structural rules) and by the full Jest/RTL
suite above, **not** by device screenshot comparison. Per the sprint's
own "no completion claim without a route-by-route visual checklist"
rule, this is reported honestly rather than claimed as visual sign-off —
a manual Expo Go/emulator pass across A1–A6 (including a short/low-width
device and large font scale, per the README's conformance gate) is still
needed before calling Workstream A visually complete.

## Remaining for a later pass

- **Manual Expo Go/emulator visual pass across A1–A6** — see above.
- **Real payment integration (a true MOB-06B-successor)**: `real` mode is
  reserved in `PaymentGatewayMode` but intentionally unimplemented — no
  Mombongo (or other) API/auth contract exists yet.
- A production-facing (or QA-only) developer control to pick a demo
  scenario visually wasn't built — `mockScenarioController.ts` exists and
  is fully testable, but nothing renders a UI over it yet.
- Producer display name: `ProducerInvoice` has `idProducteur`, not a
  joined display name — every screen shows "Réception {id}" rather than a
  producer's name, matching the reference's own "Producteur à développer"
  placeholder. Wiring a real `producteurs` lookup is a small follow-up,
  not attempted here since it wasn't in this pass's scope.
