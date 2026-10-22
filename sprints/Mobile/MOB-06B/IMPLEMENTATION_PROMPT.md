# Claude Code master prompt — MOB-06B

Implement **MOB-06B — Guided Administrator redesign and mock producer payments**.

Read the sprint `README.md`, both attached visual references, MOB-06A `RESEARCH_NOTES.md`, `IMPLEMENTATION_NOTES.md`, its amended README, and the real current code before editing. The written sprint is authoritative; the images define the required visual hierarchy and density.

## Outcome

Deliver both:

1. a genuine visual redesign of every MOB-06A Administrator route so it closely resembles the guided reference; and
2. a development-only, clearly labelled mock payment journey behind the existing capability boundary.

Do not interpret “redesign” as changing colors around the current layout. Recompose screen hierarchy, spacing, cards, guidance, imagery and CTA priority.

## Phase 1 — audit and baseline

- Inventory the exact ADMIN route list; do not rely on an assumed count.
- Trace AppShell/AppHeader/BottomNav, route guards, design tokens and Fraunces.
- Trace current producer invoice repository, reconciliation and allowed transitions.
- Trace the hardcoded-unavailable payment boundary and future-state components.
- Capture current screenshots if the environment supports it.
- Write a route-to-reference mapping and identify why the current UI differs visually.
- Confirm that no client path can set `payee` or `paiement_en_attente` in Firestore.

Proceed only within the existing single-tenant architecture; do not add an organization model.

## Phase 2 — shared guided components

Prefer reusable components such as:

- `GuidedPageIntro`
- `WhatNextCard`
- `IllustratedStatus`
- `GuidedPrimaryAction`
- `OperationalPhotoCard`
- `DemoDataBanner`
- `InvoiceComparisonCard`
- `PaymentStepIndicator`

Reuse existing tokens and shell. Do not introduce a second design system or duplicated navigation.

All authenticated pages must include the shared top header, `Mode Administrateur` badge and BottomNav. Wizard/detail headers use the existing supported pattern; `BackHeader` does not exist and must not be invented.

## Phase 3 — redesign all MOB-06A screens

Apply the README's shared hierarchy literally:

- heading;
- one-sentence explanation;
- optional status banner;
- three or four major items maximum above the fold;
- `Que faire maintenant ?` guidance;
- one dominant CTA;
- bottom navigation.

Use the exact flows and content rules in Workstream A. Preserve real business logic, real selectors, cache-first reads, quality reconciliation, raw-material-only stock meaning and task display-only limitation.

Do not add finished-goods stock, pH/Brix/temperature, task assignment, staff scoring or invented metrics.

## Phase 4 — mock payment adapter

Implement through dependency injection, not conditional logic scattered across screens.

Required conceptual interface, adapted to the repository conventions:

```ts
type DemoPaymentOutcome = "success" | "pending" | "failure" | "timeout";

interface ProducerPaymentGateway {
  getCapability(): Promise<{ available: boolean; mode: "unavailable" | "demo" | "real" }>;
  startPayment(input: { invoiceId: string; idempotencyKey: string }): Promise<PaymentResult>;
}
```

Production currently supports only `unavailable`. `real` is reserved and must not be implemented.

Mock safety:

- demo gateway is importable/instantiated only in development;
- require an explicit local development flag in addition to `__DEV__` or the repository-equivalent build guard;
- production always returns unavailable even if a public flag is mistakenly set;
- fixture invoices and simulated results stay outside Firestore repositories;
- never call invoice transition code with `payee`/`paiement_en_attente`;
- never upload a mock receipt;
- stable idempotency key per demo attempt;
- second tap returns the same in-flight/result state, not a new simulation;
- deterministic test scenarios selected through test injection or a development-only control that production cannot render.

## Phase 5 — guided payment UI

Implement every B1–B5 screen and state from the README/reference.

Every demo screen must visibly repeat:

`Données de démonstration uniquement`

and explain:

`Aucun argent n’est transféré. Aucun paiement réel.`

The success title must be `Paiement simulé avec succès`, never `Paiement confirmé` without qualification.

Use a two-step verification and confirmation flow, non-dismissible progress, duplicate-submit prevention and explicit result states.

## Responsive/accessibility requirements

- Verify low-width Android, standard phone and large font scale.
- Use ScrollView/KeyboardAvoidingView when applicable.
- Minimum 48×48 px targets and 56 px primary action.
- No information conveyed only by color.
- Accessible French labels for status icons, checkboxes, close/back and payment actions.
- Screen-reader announcement when simulation changes to processing/result.
- BottomNav and CTA must remain above Android system navigation.

## Required tests

- mock gateway unavailable outside development;
- explicit demo flag required;
- no Firestore call for simulated result;
- no real invoice status mutation;
- deterministic success/pending/failure/timeout;
- idempotency and double-tap protection;
- demo banner on every mock screen;
- correct simulated wording in every result;
- checkbox and two-step confirmation;
- progress cannot be dismissed/restarted;
- route guard ADMIN-only;
- all current invoice reconciliation/transition tests remain green;
- route-by-route AppHeader, badge and BottomNav;
- guidance card and dominant CTA on each ADMIN route;
- small-screen and large-text component behavior;
- no deterministic rules regression.

## Visual verification is mandatory

Do not finish after typecheck/tests/export.

For every actual Administrator route, produce a checklist containing:

- matching reference screen;
- heading style and size;
- explanation present;
- maximum above-fold card density;
- imagery/icon treatment;
- guidance card present;
- one dominant CTA;
- AppHeader, role badge and BottomNav;
- small-screen result;
- large-text result;
- remaining visible difference.

Capture comparison screenshots using the repository's available simulator, Expo Go, web viewport or screenshot tooling. If none is available, state that visual completion is blocked; do not claim exact design conformity based only on Jest or web export.

Run typecheck, lint, complete tests, Firebase rules tests and Expo web export. Report the known emulator cold-start failures separately from deterministic regressions.

## Final response

Return:

1. Files modified
2. ADMIN route inventory and redesign mapping
3. Shared guided components
4. Mock gateway safety proof
5. Payment state behavior
6. Tests/build results
7. Route-by-route visual comparison checklist
8. Screenshot/device evidence
9. Remaining visual differences
10. Real-payment integration blockers for a later sprint
