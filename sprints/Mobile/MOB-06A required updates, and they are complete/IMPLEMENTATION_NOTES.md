# MOB-06A — implementation notes (as built, 2026-08-29)

Companion to `RESEARCH_NOTES.md` (pre-implementation schema facts). This
records what was actually built, for MOB-06B to pick up.

## Confirmed schema used as-is (no changes)

`producteurs`, `approvisionnements`, `productions`, `qualityControls`,
`stockMP`, `ventes`, `tasks`, `users` — all read exactly per
`RESEARCH_NOTES.md`, no field additions. `calcAppro` (reception→cost
formula) is already ported to `AROM-Mobile/src/features/reception/
model.ts:77-80` — reused verbatim by `invoices/reconciliation.ts`, not
reimplemented.

**Bug fixed in passing**: `production/productionRepository.ts`'s
`getProductionHistory` hardcoded `"PRODUCTION"` as the cache-read role
instead of using its own `roleId` parameter. Every existing caller only
ever passed `"PRODUCTION"` anyway (zero behavior change for them), but this
silently broke `quality/qualityRepository.ts`'s `getLotsToControl(uid,
"ADMIN")` — the exact reuse this sprint depends on for A4. Fixed and locked
in with a regression test (`productionRepository.test.ts`).

## New Firestore domain: `producerInvoices/{id}`

Schema, rules (`isValidProducerInvoice`, `isValidInvoiceTransition`), and
rules tests are in `AROM-Backend/firestore.rules` and
`AROM-Backend/tests/rules.test.mjs` (21 new tests, all passing across two
clean emulator runs — see "Test results" below).

- **Business status** (`statut`): `recue → a_verifier → approuvee |
  contestee`. `paiement_en_attente`/`payee`/`annulee` exist in the enum for
  forward-compatibility but are **unreachable from any mobile code path**
  — enforced both in `firestore.rules`'s `isValidInvoiceTransition` and in
  `invoices/model.ts`'s client-side mirror. No manual mark-as-paid exists
  anywhere.
- **Local sync status**: separate concept, derived from the transition's
  `MutationEnvelope` (`invoices/invoiceStatus.ts`) — kept structurally
  distinct from business status via two separate badge components
  (`StatusBadge` vs `BusinessStatusBadge`), mirroring quality's own
  `StatusBadge`/`DecisionBadge` split.
- **No invoice-creation UI exists** — ingestion is out of scope per the
  README. Mobile only reads real invoices and transitions
  `a_verifier→approuvee`/`a_verifier→contestee` via `invoices/
  invoiceSync.ts`'s `submitTransition`, which replicates `quality/
  qualitySync.ts`'s transaction pattern (fresh `tx.get()` re-read,
  idempotent no-op if already at target, thrown non-retryable conflict
  otherwise) — a single-doc status precondition instead of a sibling-doc
  existence check, since this is always an update to one already-existing
  invoice.
- **Estimate vs. invoice**: `invoices/reconciliation.ts`'s
  `getEstimatedPayables` derives a `Montants estimés` figure from
  receptions with no matching `producerInvoice.approvisionnementId`, via
  `calcAppro` — always labelled "Estimation," never auto-converted into an
  invoice record.

## Payment capability boundary (why it's disabled)

`payments/producerPaymentCapability.ts`'s `getProducerPaymentCapability()`
is the **only** function in the app allowed to answer "can we pay a
producer." Its body is a literal hardcoded
`{ available: false, reason: "Paiement indisponible pour le moment" }` —
no flag, no env var, no branch. This is deliberate: MOB-06B's job is to
replace the function body wholesale (a visibly reviewable diff), not flip
a bit that could be flipped by accident. `Payer la facture` always renders
disabled with that message in `producer-invoice/[id].tsx`. No code path
anywhere enqueues, infers, or fabricates a `paiement_en_attente`/`payee`
transition — confirmed by grep, not just by review (see "Manual
verification" below).

`payments/components/PaymentFlowStates.tsx` holds the presentational
confirm/processing/pending/success/failure/unknown-timeout states the
future real flow will need — exercised only by its own component test,
**never imported from `src/app`** (test-enforced).

## Dashboard architecture

ADMIN was forked out of the shared `RoleHomeScreen` (see README Amendment
4) into `src/features/admin/`:

- `useAdminDashboardData.ts` — cache-first loader for ADMIN's 9 sync
  categories (extended in `sync/syncPlan.ts`: added `qualityControls`,
  `stockMP`, `producerInvoices` to ADMIN's arm, which previously omitted
  them entirely).
- `adminOperations.ts` (A3), `adminPriorities.ts` (A4, reuses
  `qualityRepository.getLotsToControl` unmodified for the quality
  reconciliation), `adminActivity.ts` (A2), `adminTeamActivity.ts` (A5) —
  pure selectors, each with direct unit tests (not indirected through a
  screen render, since ADMIN no longer goes through `RoleHomeScreen`'s
  `buildMetrics`).

**Screen map**: A1 → `home.tsx` (ADMIN branch) →
`AdminOverviewScreen.tsx`. A2 → `activity.tsx` (ADMIN branch). A3 →
`admin-operations.tsx`. A4 → `admin-priorities.tsx`. A5 → `team.tsx`
(extended in place — it already existed and was already ADMIN-only). A6 →
`admin-priorities/[id].tsx` (re-derives the item from `buildPriorities`
against fresh cache — priorities aren't a stored collection). B1 →
`producer-invoices.tsx`. B2/B3 → `producer-invoice/[id].tsx` (one file,
branches on `statut`, mirroring `quality-control/[id].tsx`'s pattern).

`ROLE_HOME_CONFIG.ADMIN.actions` grew from 2 to 4: `priorities` now routes
to `/admin-priorities` instead of `/tasks` (the bottom-tab `Tâches` still
points at the unchanged, shared `/tasks` — no per-person task assignment
was added, per README Amendment 3), plus new `operations` and `invoices`
entries.

## What was intentionally not built

- **Overdue-task priority**: `tasks/{id}` has no due-date field
  (`RESEARCH_NOTES.md` item 7) — inventing a threshold to derive "overdue"
  would violate the sprint's own anti-fabrication rule. Omitted rather than
  guessed.
- **Real invoice ingestion, MOB-06B payment execution, manual mark-as-paid,
  finance/GL/tax reporting, staff performance scoring, MOB-05C quarantine
  resolution** — all explicitly out of scope per the README.
- Visual polish is functional (cards/lists/chips using existing theme
  tokens) rather than the fully photo-led treatment the reference images
  show — `SummaryCard`'s icon-tile fallback path is used rather than
  building bespoke per-module photography, since no such assets exist yet
  and the README states no remote-placeholder dependency is allowed.

## Test results (as of 2026-08-29)

- AROM-Mobile: **81 test suites / 677 tests, all passing** (`bun run
  test`), including full MOB-01–MOB-05B regression. `bun run typecheck`
  and `bun run lint` both clean. `npx expo export --platform web`
  succeeded; all 8 new routes appear in the static route list
  (`admin-operations`, `admin-priorities`, `admin-priorities/[id]`,
  `producer-invoices`, `producer-invoice/[id]`, plus the extended
  `home`/`activity`/`team`).
- AROM-Backend rules suite: run twice via `firebase emulators:exec
  --only firestore,storage "npm test"` against a locally running
  emulator. **Both runs: 74/77 passing, identical 3 failures both times**
  — all three are the first-upload-per-`describe`-block Storage tests
  (`approvisionnements`/`producteurs`/`qualityControls` evidence photo
  suites), matching the documented pre-existing cold-start race exactly.
  All 21 new `producerInvoices` tests passed on both runs; no Firestore
  test failed on either run.

## Manual verification performed (not simulator/device — code-level only)

- `grep -rn "PaymentFlowStates" src/app` → empty.
- `grep` for `"payee"`/`"paiement_en_attente"` literals outside
  `invoices/model.ts`'s own enum definition → empty.
- Every new/extended screen renders one `AppShell` at its root (title,
  `onBack` where applicable, `roleLabel: ROLE_CONFIG.ADMIN.badgeLabel`,
  `activeTab`, `roleId: "ADMIN"`) — structurally guaranteed by reusing
  `AppShell`, not just tested per-screen; each screen's test additionally
  asserts the "Mode Administrateur" badge text renders and (for
  ADMIN-only routes) that a non-ADMIN account is redirected to
  `/role-select`.

## Not done — needs a real device/simulator

Bottom-nav safe-area clearance, actual touch-target sizing on a physical
screen, and any true offline/airplane-mode behavior were not verified on
a simulator or device this session — only via Jest/RTL and the web
export. Flagging per the sprint's own instruction not to claim
device verification that wasn't performed.

## Open MOB-06B decisions

- Real invoice ingestion mechanism: webhook vs. Cloud Function vs. an
  AROM-Production admin tool — not decided; `producerInvoices`' rules
  support any of them once a service-account/admin-write path exists.
- Real payment adapter behind `getProducerPaymentCapability` — the
  function signature (`{ available, reason }`) is the contract MOB-06B
  must satisfy against a real backend check.
- Idempotency/reconciliation for the real payment flow: `referenceExternePaiement`
  on `ProducerInvoice` is reserved (opaque to mobile) for this purpose but
  nothing writes it yet.
- Receipt/callback handling for `paiement_en_attente`/`payee`/`annulee`
  transitions: rules already forbid these from any client-writable state,
  so MOB-06B's backend-authoritative writer will need its own Admin-SDK
  path (rules-bypassing) or a dedicated Cloud Function service account.
