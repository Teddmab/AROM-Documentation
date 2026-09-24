# AROM Mobile Completion Plan

This folder is the execution guide for finishing AROM Mobile after MOB-01 through MOB-11.

## How to use it with Claude in VS Code

1. Copy this entire folder into the root of the repository as `docs/completion-plan/`.
2. Open Claude from the repository root so it can inspect `AROM-Mobile`, `AROM-Backend`, `AROM-Production`, and `AROM-Documentation`.
3. Start with `prompts/01-documentation-and-flags.md`.
4. Give Claude only one sprint prompt at a time.
5. Review Claude's audit findings before allowing edits when the prompt contains a decision gate.
6. Require tests and the completion report before moving to the next sprint.
7. Commit each completed sprint separately.

Do not paste the full PDF into Claude as one giant prompt. The PDF is a product and design reference; these Markdown files are the implementation instructions.

> **Copy note.** This folder mirrors `AROM-Mobile/docs/completion-plan/` as merged in AROM-Mobile PR #21 (2026-09-24). Paths below starting with `docs/` are relative to the `AROM-Mobile` repository.

## Unified-app decisions (owner, 2026-09-24)

Recorded in full in `AROM-Mobile/docs/simplification/UNIFIED_APP_AUDIT.md`:

1. One shared structure — Accueil, À faire, Historique and Plus — identical for every account.
2. Roles remain authorization context only: `role`/`poste` grant capabilities; they never select a different app, home or tab bar. No role picker, no working modes, no mode switch.
3. Sections an account cannot use are hidden, not shown empty, locked or greyed.
4. The market action label is "Acheter des fruits"; "Mombongo" remains only a small source label.
5. Automatic stock movements and automatic invoice approval are deferred; both stay manual.
6. Existing contracts, capabilities, offline behaviour, audit data and deep-link redirects must remain intact.

## Decisions already made

- Keep one application with one shared structure (Accueil, À faire, Historique, Plus) for every account. Roles are authorization context only — they grant capabilities, never a separate mode or home (superseded 2026-09-24, see `docs/simplification/UNIFIED_APP_AUDIT.md`).
- Keep the existing Firebase, cache-first reads, drafts, and ordered mutation queue.
- There is no role picker and no mode switch; sections an account cannot use are hidden (see `docs/access/CAPABILITIES.md`).
- Add near-real-time freshness by refreshing on foreground and key-screen focus; do not replace the data layer.
- Hide Google and Facebook login until their client IDs are configured.
- Require scoped staff accounts and close Firestore shape-validation gaps.
- Protect every wizard exit.
- Simplify reception around the producer and sales around products.
- Treat external customer orders as a new `partnerOrders` domain, not producer invoices.
- Do not enable partner orders until finished-product stock and payment ownership are reliable.

## Execution order

| Order | Prompt | Outcome |
|---|---|---|
| 1 | `01-documentation-and-flags.md` | Documentation and release configuration reflect reality |
| 2 | `02-freshness.md` | Cross-device data appears without manual resync |
| 3 | `03-security.md` | Staff accounts and Firestore writes are safely scoped |
| 4 | `04-navigation-and-drafts.md` | Back, drafts and deep links are predictable (mode switching was removed — see `docs/access/CAPABILITIES.md`) |
| 5 | `05-reception-and-production.md` | Core field journeys are simpler and interruption-safe |
| 6 | `06-sales.md` | Sales are product-first with safe automatic traceability |
| 7 | `07-admin.md` | ADMIN pages have distinct operational purposes |
| 8 | `08-finished-product-stock.md` | Sellable stock and reservations are authoritative |
| 9 | `09-partner-orders.md` | Signed order-to-fulfilment pilot, only after approval gates |
| 10 | `10-field-validation.md` | Three uncoached workflows and release decision |

## Stop conditions

Stop and ask for a decision when:

- payment ownership between AROM and Mombongo is still unclear;
- the official formats or prices are not confirmed;
- a proposed metric cannot be derived reliably from the existing schema;
- Firestore rules require a breaking migration of existing production accounts;
- finished-product inventory cannot prevent concurrent overselling;
- a prompt conflicts with newer code or a signed integration contract.

## Global rules for every sprint

- Inspect the current code, schemas, routes, tests, and repository instructions before editing.
- Reuse existing architecture and dependencies.
- Never hard-code names, prices, quantities, counts, dates, thresholds, or statuses from mockups.
- Keep business state separate from device synchronization state.
- Preserve offline cache, drafts, mutation ordering, idempotency, and role authorization.
- Use plain French in the interface. Never show Firestore, payload, mutation, webhook, or server-conflict language.
- Use icon plus text for status; never color alone.
- Keep critical touch targets at least 48 by 48 px and primary actions at least 56 px high.
- Add loading, empty, offline, stale, partial, error, and retry states where relevant.
- Run typecheck, lint, the complete test suite, and Expo web export.
- End with files changed, behavior changed, tests run, remaining risks, and the next recommended prompt.

