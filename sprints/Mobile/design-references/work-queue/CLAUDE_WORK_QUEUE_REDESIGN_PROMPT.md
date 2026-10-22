# Claude prompt — replace Consignes with a useful role-aware work queue

Implement the AROM-Mobile work-queue redesign using the supplied mockups and transparent assets.

## Investigate before editing

Inspect `tasks.tsx`, `BottomNav`, `syncPlan.ts`, task models/repositories, Firestore task rules, route ownership, existing task completion handlers, ADMIN priorities, and every task producer in Mobile and AROM-Production.

Produce a matrix of each real task stage showing: creator, owner role/poste, authorized reader, authorized actor, destination route, completion condition, and whether it is actionable or follow-up only.

## Product decision

- Replace the bottom-tab label `Consignes`.
- ADMIN sees `À traiter`: decisions and exceptions ADMIN can genuinely handle.
- Operational roles see `À faire`: only actions the current poste can execute now.
- `En suivi` is a secondary collapsed section for information only.
- Never mix another poste's work into the actionable list.
- Client filtering is not the security boundary; keep Rules aligned.

## Required behavior

1. Show an item in `À faire`/`À traiter` only when the current account is authorized and a real destination/action exists.
2. Use human titles and context. Technical IDs are secondary. Raw email addresses and internal stage names are never card titles.
3. One dominant CTA per actionable card.
4. Completing an action removes it from the active queue and refreshes the list without restarting.
5. Follow-up records go to `En suivi` or Activity.
6. Empty state appears only when the active queue is truly empty.
7. ADMIN must reuse the canonical priority/task logic where possible; do not build a second priority engine.
8. Team invitations are ADMIN-only and must never appear for Production or Collection.
9. Preserve offline cache, honest freshness states, draft protection, actor uid, route authorization, and role/poste scoping.
10. Do not expose actions that Firestore Rules will reject.

## Visual implementation

Reference mockups:
- `docs/design-references/work-queue/admin-a-traiter.png`
- `docs/design-references/work-queue/production-a-faire.png`

Runtime assets:
- `assets/images/work-queue/task-quality-control.png`
- `assets/images/work-queue/task-invoice-review.png`
- `assets/images/work-queue/task-order-confirmation.png`
- `assets/images/work-queue/task-reception-transform.png`
- `assets/images/work-queue/task-production-quality.png`
- `assets/images/work-queue/task-stock-outgoing.png`

Verify alpha, filename/content matching and optimize without upscaling. Cap longest edge near 384–480px. Use one centralized `require()` map and `expo-image` with `contentFit="contain"`. Decorative images are hidden from accessibility. Replace them with compact code-native glyphs under short-height or large-font crowding.

Do not copy illustrative mockup numbers into production.

## Role intent

ADMIN `À traiter` may include real actionable states such as quality waiting, quarantine resolution, invoice verification/payment failure, order confirmation, synchronization conflict, stock inconsistency and ADMIN-only team approval.

Production `À faire` may include reception ready for transformation, production draft/resume, quality control, quarantine resolution when authorized, and assigned stock movement. Do not assume each example exists: derive only from real workflows.

Collection and Sales need the same contract but their own real actions. Do not populate them with Production examples.

## Avoid over-engineering

- Reuse the existing `tasks` collection if its semantics are sound.
- Reuse ADMIN priorities where they already represent the same exception.
- Do not introduce a workflow engine, scheduler, polling loop or new global state framework.
- Do not create synthetic task documents merely to make the mockup look populated.
- If an existing task has no authorized actor or route, classify and repair/remove that task producer rather than displaying a dead card.

## Required tests

Cover role/poste stage ownership, Rules/query compatibility, actionable-vs-follow-up separation, team-invite isolation, CTA destination, unauthorized route rejection, offline cached queue, partial refresh failure, immediate removal after completion, empty-state correctness, same-id collision safety if multiple sources merge, accessibility, compact height, large font scale and unchanged non-admin authorization.

Use `arom-qa` only for live verification. Capture ADMIN, Production, Collection, Sales, empty, offline/partial, compact and large-font states. Do not deploy production and do not start another sprint.

Stop and report if a task has no authoritative owner, completion rule, authorized route or trustworthy source.

Return the task-stage matrix, exact behavior retained/removed, files changed, Rules changes if genuinely required, asset sizes, screenshot paths, test totals and unrelated changes left untouched. Do not commit until the full suite, typecheck, lint and Expo export pass.
