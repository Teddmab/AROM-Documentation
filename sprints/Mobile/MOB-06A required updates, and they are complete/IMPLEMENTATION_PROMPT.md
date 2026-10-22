# Claude Code master prompt — MOB-06A

Implement `MOB-06A — Administrator oversight and producer payables foundation` in AROM-Mobile and only the minimum necessary shared contracts/rules in AROM-Backend and AROM-Production.

Read the attached sprint `README.md` (including its "Amendments after
research" section) and `RESEARCH_NOTES.md` completely before editing. Use
both reference images for visual hierarchy. The README is authoritative
over image text and sample values.

## Phase 1 — evidence report (complete)

The evidence report this phase called for has already been produced and is
recorded in `RESEARCH_NOTES.md` in this folder, covering: ADMIN home, role
guards, AppShell, AppHeader (BackHeader is dead — see README Amendment 2)
and BottomNav; real Firestore entity names/fields for producers, receptions,
productions, `qualityControls`, `stockMP`, sales, tasks and users; existing
cache-first reads and initial-sync categories (and the ADMIN sync-plan gap
to close for `qualityControls`/`stockMP`); real price/quantity/cost
calculations for reception payables (`calcAppro` in
`AROM-Production/src/lib/erp/engine.ts`); ADMIN permissions in rules and the
confirmed absence of any organization-isolation concept (README Amendment
1); existing activity/audit timestamps and actor fields, including the
confirmed absence of `lastActivity`; confirmed absence of stock thresholds,
invoice records, payment records and external references; and the
implemented MOB-05B quality contract's dependency chaining and
one-control-per-lot transaction in `qualitySync.ts`, which must be reused
as-is for invoice/quality reconciliation rather than reimplemented.

Read `RESEARCH_NOTES.md` fully before writing any code — it separates
confirmed facts from gaps and cites exact files/lines. Proceed with
implementation using confirmed contracts. Do not pause merely because
invoice payment is unavailable; implement the honest capability-gated state
defined below.

## Scope order

1. Reusable ADMIN aggregation selectors/hooks from real cached collections, including `qualityControls`.
2. Administrator overview, operations, priorities, team activity and priority detail routes.
3. AROM-owned producer invoice/payable domain foundation only if the real schema can support it safely.
4. Invoice list, detail, validation/dispute and payment-unavailable state.
5. Target payment UI state components may be built/tested behind a disabled capability flag, but do not connect or simulate transfers.

## Administrator dashboard

Replace the current three-card ADMIN home while preserving the shared role-home architecture.

Build:

- `Vue d’ensemble`
- `Activité du jour`
- `Suivi des opérations`
- `Priorités`
- `Activité de l’équipe`
- `Détail d’une priorité`

Every metric and priority must be derived by a pure, tested selector from real data. Do not hard-code example values. Do not treat missing data as zero when that would mislead; show unavailable/last-sync state.

Priority derivation may include, only when supported:

- producer invoice/payable requiring review;
- lot awaiting quality control;
- quarantined or rejected quality decision requiring ADMIN awareness (read-only until MOB-05C);
- failed or pending quality-control synchronization without double-counting the same lot;
- overdue task;
- failed synchronization;
- raw-material stock warning only when a real threshold exists;
- sales/operational exception with a real status/date rule.

Do not invent staff productivity. Show only the latest verifiable business event attributed to a user.

Use `qualityControls.staffUid`/`responsable` only according to the real contract. Reconcile productions against synchronized and locally queued quality controls so an offline control does not incorrectly appear as a second “lot awaiting control” priority.

There is no finished-goods stock collection. Never derive a finished-goods availability total, write a stock movement, or label `stockMP` as finished products. Quality decisions supply status/oversight information only. A quarantined lot can be viewed but cannot be released or re-controlled in this sprint.

## Producer invoice foundation

The UI name is always `Factures producteurs`. Never display Mombongo or external-provider details.

First determine whether invoices exist. If absent, add a normalized AROM contract only after mapping existing conventions. Do not guess collection names in advance.

Minimum conceptual data, adapted to real naming conventions:

- stable invoice ID;
- organization scope;
- producer ID;
- linked reception/approvisionnement ID;
- supplier invoice reference only when provided;
- invoice date and optional due date;
- currency;
- line/quantity/unit-price/cost information sufficient to explain the total;
- total amount;
- business status;
- dispute reason when applicable;
- creation/update/audit fields;
- optional future external reference opaque to the UI.

Do not store provider credentials, access tokens or secret payloads in Firestore/mobile.

If there is no actual invoice ingestion yet:

- show invoice empty state;
- optionally derive `Montants estimés` from reception data only if the real ERP calculation is authoritative;
- never convert an estimate into an invoice automatically;
- never create fake invoice records to populate the dashboard.

## Review and validation

Implement invoice list/detail only for real records. Reconcile invoice values with the linked reception and surface supported differences. Allow ADMIN to validate or dispute through narrowly scoped, tested transitions. Do not allow manual `Paid` status.

Use transaction/idempotency patterns consistent with MOB-04/MOB-05, specifically the dependency-chaining/one-control-per-lot transaction pattern already implemented in `AROM-Mobile/src/features/quality/qualitySync.ts`. Prevent duplicate validation mutations; there is no organization concept to isolate against (README Amendment 1) — gate by role/poste (`isAdmin()`) as every other collection already does.

## Payment capability boundary

Create a provider-neutral capability interface or backend contract boundary; do not integrate an external provider in this sprint.

The mobile UI asks the backend whether payment execution is available. With current configuration it must honestly show:

`Paiement indisponible pour le moment`

The `Payer la facture` action must not execute or pretend to succeed. Future MOB-06B will connect this boundary to a real backend adapter.

You may create presentational/testable states for confirmation, processing, pending, success and failure behind fixtures used only in component tests or Storybook-like development previews. Never expose those fixtures in production runtime.

## Offline behavior

- Dashboard and invoice list are cache-first with last-sync time.
- Invoice validation/dispute may queue offline only if its conflict policy is explicit and safe.
- Payment execution must never queue as an ordinary offline mutation.
- When offline, disable payment and explain connectivity is required.
- Never infer `Paid` from a local mutation.

## Shared chrome — mandatory

Reuse existing AppShell and design tokens.

Every authenticated route must show:

- AppHeader or BackHeader;
- `Mode Administrateur` badge;
- BottomNav with `Accueil`, `Tâches`, `+`, `Activité`, `Plus`;
- top and bottom safe-area protection.

Modals/bottom sheets inherit clear context and return to a screen that retains the shell. Ensure bottom nav never overlaps a card, CTA or Android system navigation.

## Design

- Background `#FBF7EB`, primary `#005A2B`, gold/yellow accent.
- Existing Fraunces token for headings; do not bundle a duplicate font.
- White rounded cards, restrained shadow, clear status chips.
- Large photo-led module cards where existing assets are available; no remote placeholder dependency.
- Minimum 48px targets and short French copy.
- Loading, empty, offline, partial-data, error and permission states.
- Red only for true critical/action failure states.

## Security and data integrity

- ADMIN UI checks do not replace rules/backend authorization.
- Gate every invoice and aggregate query by role/poste (`isAdmin()`), matching every other collection — there is no organization concept in this system to isolate against (README Amendment 1).
- Do not broaden existing blanket access.
- Validate field shape and allowed status transitions.
- Financial totals remain server-authoritative where a backend calculation exists.
- Never log invoice-sensitive data or future payment tokens.

## Tests

Add tests for:

- all dashboard selectors and missing/partial cache;
- productions awaiting control versus synchronized/queued `qualityControls`;
- quarantined/rejected visibility with no resolution action;
- absence of invented finished-goods stock totals or writes;
- priorities ordered by real severity rules;
- activity attribution without fabricated actors;
- ADMIN route guard and cross-role denial;
- role/poste rules gating (no organization concept exists — see README Amendment 1);
- invoice empty state and estimated-payable distinction;
- invoice/reception reconciliation;
- allowed validation/dispute transitions;
- payment capability unavailable and offline;
- no manual paid transition;
- header, role badge and bottom nav on every route;
- bottom safe area/CTA clearance;
- accessibility labels and large-text layout;
- regression of all prior sprints.

Run repository-equivalent typecheck, lint, Jest, Firebase emulator rules tests and Expo web export. The Firebase rules suite has a known pre-existing first-upload Storage-emulator cold-start race. Run sufficient clean repetitions to identify deterministic regressions, report each exact result, and do not weaken rules or falsely call a failing run green. Do not claim simulator/device verification unless performed.

## Documentation

Update the data model and sprint notes with:

- confirmed existing schema;
- any normalized invoice model introduced;
- estimate-versus-invoice distinction;
- payment boundary and why it is disabled;
- open MOB-06B integration decisions: ingestion API/webhook, authentication, callbacks, idempotency, status reconciliation and receipts.

## Final response

Return:

1. Files created/modified
2. Confirmed schema and gaps
3. Dashboard data sources and priority rules
   - explicitly identify how `qualityControls` and raw `stockMP` are represented
4. Invoice model/status transitions
5. Payment capability behavior
6. Security/offline behavior
7. Tests and build results
8. Header/bottom-nav verification by screen
9. Remaining MOB-06B blockers
10. Manual device checks still needed
