# Automation engine — design (proposed, not yet built)

Status: **design only**. Nothing in this document is implemented. Written to
scope the "AROM ADMIN automation" mobile screens (`Paramètres
opérationnels`, `Réglages de production`, `Automatisations`) before any
code — those screens depict background automations, a finished-goods stock
ledger, and an operations audit log, none of which exist in the current
system. See `AROM-Mobile`'s screen mockups
(`assets/images/admin-automation/`) for the visual target these
collections/functions need to support.

Four decisions already made (2026-09-15, with the client/product owner):

1. **Execution model: a Cloudflare Worker Cron Trigger, not Firebase Cloud
   Functions.** Reuses AROM-Production's existing Workers deploy pipeline
   and the precedent already set by the Mombongo integration (server-side
   privileged logic lives in AROM-Production's own `/api/*` routes, not a
   second Firebase runtime) — `roadmap.md` has twice deliberately deferred
   introducing Cloud Functions to avoid a second billing/ops surface; this
   avoids it a third time. Tradeoff accepted: automations run on a short
   polling interval (proposed: every 2 minutes), not instantly on write.
2. **"Delivery → create the sale once" stays a manual dashboard action.**
   AROM-Production's existing "mark Livrée" flow
   (`OrdersCard`/`dashboard.tsx`, see `data-model.md#order--ventes-bridge`)
   is untouched — already idempotent (deterministic `ventes` doc id), no
   reason to touch working, tested code. The mobile monitor screen logs it
   as one of the visible automation types for consistency, not because its
   trigger changed.
3. **`stockPF` only counts juice after quality-control release**
   (`decision: "liberer"`), never at production-save time. The mockup's own
   worked example ("AROM proposera 170 L" the instant a production is
   saved) is now known to be illustrative, not literal — the real trigger
   for a `stockPF` Entrée row is a `qualityControls` doc reaching
   `decision: "liberer"`, not a `productions` doc being created. **Resolved
   and implemented** (finished-stock batch, 2026-09) — this is no longer an
   open question (the §"Open question for the client" note under the
   `stockPF` schema below is superseded; see the finished-stock section
   further down for the shipped design). This changes step 1 of the cron
   job below accordingly (see the updated query).
4. **`perteHabituellePct` is not a new field** — it's
   `config/parametres.tauxPertesMax`, read and displayed by the mobile
   settings screen under its French mobile label. One source of truth;
   the web ERP's existing KPI formulas and the mobile settings screen can
   never drift apart on this number.

## Execution boundary: confirmation-time vs. the Worker

Added after shipping the first real piece of this document (ADMIN
production-settings integration, mobile-only): a rule for which
"automation" consequences belong where, made concrete now that one exists
end-to-end rather than only in design.

1. **Immediate business consequences happen at confirmation, through the
   authoritative write path already producing the record — never by
   waiting on a cron tick.** The yield-variance ADMIN priority is the
   worked example: it is not written by any job. `productions/{id}`
   already carries everything needed to judge it (`expectedVolumeJusL`,
   `parametresSnapshot.ecartAcceptablePct`) the moment the mobile wizard's
   `syncOne` transaction commits that document, and `adminPriorities.ts`
   recomputes the priority live, from that same document, on every read —
   no separate write, no polling delay, no window where a real "explain
   this" case is invisible to ADMIN because a 2-minute-interval job hasn't
   run yet.
2. **The Cloudflare Worker (§"The Cloudflare Worker cron job" below) exists
   for reconciliation, safe retries, and detecting *missed* consequences —
   not for producing the first, normal-path consequence of an event.**
   Concretely: once `stockPF`/reservation ship, "a `qualityControls` doc
   reached `decision: liberer`" should ideally trigger its `stockPF` write
   through the same kind of confirmation-time path (a Cloud Function or an
   equivalent synchronous step), with the Worker's poll only catching
   whatever that primary path failed to complete (a crashed request, a
   partial write) — not standing in as the only path. Decision #1 (Worker,
   not Cloud Functions) was about *where privileged server logic lives*,
   not a license to make every consequence poll-interval-latent by
   default.
3. **Normal stock accuracy must not wait for a cron interval.** If a
   future batch finds that a stockPF/reservation consequence can only be
   made correct by putting the *first* write of it behind the Worker's
   poll (rather than behind a confirmation-time path with the Worker as
   its reconciler), that is the exact "stop and report the integrity
   risk" case — the same one this document's client-side production-
   settings work never hit, because `adminPriorities.ts` could compute
   its priority from already-authoritative client-written data with no
   privileged step in between.

## New Firestore collections

### `stockPF/{id}` — finished-goods stock ledger

**Shipped** (finished-stock batch, 2026-09) — the first authoritative
slice: QC-release-driven `"Entrée"` rows only. `"Sortie"`/`"Ajustement"`
rows (reservation, sale deduction, manual correction) are reserved shape,
not yet written by anything — see "Not yet implemented" below.

Deliberately narrower than the original proposal below it: `relatedId`
(one ambiguous foreign key) is replaced with two explicit ones
(`productionId`, `qualityControlId`), since every real row needs to prove
both — a `stockPF` row's own security-rule validation reads both
referenced documents (see `AROM-Backend/firestore.rules`'
`isValidProductionStockPFCreate`).

| Field | Type | Notes |
|---|---|---|
| `id` | string | Deterministic — `` `PF-IN-${productionId}-${qualityControlId}-${format}` `` — never `Crypto.randomUUID()`. One idempotent row per (production, releasing quality control, format); a retry addresses the exact same document instead of creating another, and Firestore's own create-only semantics (no `update`/`delete` rule) reject a second attempt outright. See "Idempotency" below. |
| `date` | string (ISO date) | The releasing `qualityControls` doc's own `date` — "when this stock became available," not "today" (a synced-later replay must not shift it). |
| `format` | `"500ml"` \| `"330ml"` \| `"300ml"` | Bottle-format token, no space — matches `Production.q500`/`q330`/`q300`'s own field-name convention more directly than `ventes`/`products`' own `"500 ml"` (with space). A deliberate, scoped exception for this one new collection, not a retroactive rename of `ventes`/`products` — noted here so the two conventions never get silently assumed identical. |
| `type` | `"Entrée"` (only value written so far) | `"Sortie"` / `"Ajustement"` are reserved for reservation/sale-deduction/correction, not yet implemented — see below. |
| `quantite` | number (bottles, not kg — distinct unit from `stockMP`) | Exactly the production's own `q500`/`q330`/`q300` field for this format — never derived from `quantiteConformeL` (a litres figure) or any other computed value. Always `> 0` — a zero-quantity format never gets a row at all. |
| `source` | `"production"` (only value written so far) | `"reservation"` / `"reservation-release"` / `"manuel"` are reserved, not yet implemented. |
| `productionId` | string | The `productions/{id}` this row's quantity came from. |
| `qualityControlId` | string | The `qualityControls/{id}` whose `decision: "liberer"` authorized this row — this **is** the "which release generation" key: a different releasing control (a fresh release vs. a quarantine's resolution) always produces a different `id`, never overwrites the other. |
| `createdAt` | string (ISO timestamp) | |
| `createdByUid` | string, **required** | The uid of the authenticated actor whose sync transaction wrote this row — **required, not optional** (authorization audit, 2026-09: a prior version left this optional, which is no longer acceptable). `firestore.rules` additionally requires `createdByUid == request.auth.uid` on every create, so this can never be a claimed identity — only the literal caller's own uid. A future historical backfill needs its own trusted service/audit identity through a separate path (see the eligibility-report script's own top-of-file comment) rather than making this field optional again. |

**Write path: the existing `qualityControls` sync transaction, not the
Worker.** `AROM-Mobile/src/features/quality/qualitySync.ts`'s `syncOne`
already runs a Firestore transaction that reads the source `productions/{id}`
doc and writes the `qualityControls/{id}` doc — extended to also write this
production's `stockPF` "Entrée" row(s) in that *same* transaction, only
when `decision === "liberer"` (a fresh release or a quarantine resolved to
release — both go through this one function). Offline: the `qualityControls`
mutation queues exactly as before; `stockPF` rows are only ever written once
that mutation actually syncs online, atomically with it — never a separate
step, never dependent on the Worker's poll. See "Execution boundary" above,
point 1 ("immediate business consequences happen at confirmation, through
the authoritative write path already producing the record").

**Idempotency, including the lost-ack case** (authorization/idempotency
audit, 2026-09). The transaction's own existing idempotency check
(`if (existing.exists()) …` on the `qualityControls` doc) makes a replayed
`syncOne` for an already-committed control a no-op — but existence alone
used to be trusted blindly. It no longer is: when the control doc already
exists, `syncOne` now runs an explicit preflight before reporting success —
it verifies the stored control's content genuinely matches what this retry
intended to (re-)write, and (if the release granted stock) that every
expected `stockPF` row also already exists with matching content. Only
when everything matches is this reported as a successful, idempotent
no-op — covering the exact "transaction committed, client never
received/persisted the ack, same mutation retried" scenario without ever
re-writing anything. A mismatch (a genuinely different document at that
id, or a missing/inconsistent `stockPF` row) raises a real, permanent
conflict instead — surfaced the same way a competing-control conflict
already is, never silently accepted and never overwritten. The
deterministic `stockPF` id remains a second, independent layer underneath
all of this: Firestore rules grant this collection `create` only
(`update`/`delete` are unconditionally denied), so even a write reaching
this collection through some other path can never overwrite or duplicate
an existing row — attempting to "create" an already-existing id is
evaluated as an `update` by Firestore itself and is therefore always
denied, which is exactly why the preflight above has to live in the
application layer: a blind retry write would simply be rejected by rules
rather than told "this already matches, you're done."

**Read and create authorization** (authorization audit, 2026-09: `create`
used to mirror `qualityControls`' own then-broader predicate — reported
as unacceptable and corrected; a legacy predicate copied from a sibling
collection is not an acceptable way to define this ledger's own security
model). **Read**: `isAdmin()`, `isProductionStaff()` (Directeur de
Production), and `isCommercialStaff()` (Chargée de Commercialisation —
needs to know what's sellable) — no `isUnscopedStaff()`/"Personnalisé"
fallback, since this collection has no pre-sprint-17 grandfather clause to
honor, same reasoning as the `tasks` collection's own security-hardening
pass. **Create**: `isAdmin()` or `isProductionStaff()` only — the exact
recognized QC-release actor set, nothing broader. Every other actor is
denied: unscoped/"Personnalisé" staff, a missing/empty/unknown poste,
Agent de collecte, Chargée de Commercialisation, a partner, and an
inactive account. `qualityControls`' own `create`/`update` predicate was
narrowed to match this exactly (see that collection's own section below)
specifically so a QC release and its `stockPF` side effect are always
authorized by the identical actor set — an account that can start a
release can always finish it, and an account that can't start one never
gets a confusing partial failure at the stock step instead.

**AROM-Production's web dashboard does not read this ledger.** Its
"Stock produits finis" figure is a QC-gated *projection*, computed from
`productions` + `qualityControls` using the same head-control/`liberer`
gating rule this collection's own writer uses — not a read of the
`stockPF` collection itself. That projection is equivalent to summing this
ledger only for what's currently supported: production receipts. It must
migrate to real aggregation over `stockPF` once order fulfilment (Sortie),
damage, or manual corrections exist as real rows here — see "Next
lifecycle" below. Do not describe the web dashboard's figure as reading
this ledger, or as a permanent single source of truth, until that
migration happens.

**Not yet implemented, on purpose** (do not build ahead of real need):
order reservation, sale deduction, cancellation release, expiry, forecasts,
margins, the automation monitor screen, a generic inventory engine. The
Worker cron job below remains a *future reconciliation* mechanism only —
nothing in this batch depends on it running, and normal stock visibility
never waits for its poll interval.

### `automationRuns/{id}` — the operations/audit log the monitor screen reads

| Field | Type | Notes |
|---|---|---|
| `id` | string | |
| `type` | `"production-stock"` \| `"order-reservation"` \| `"order-cancellation-release"` \| `"delivery-sale"` | The four automation kinds in the mockup |
| `status` | `"success"` \| `"failed"` \| `"needs-review"` | `"needs-review"` is the CMD-1048 case — a real, actionable, retryable problem, distinct from a transient technical failure |
| `relatedCollection` / `relatedId` | string | e.g. `"orders"` / `"CMD-1048"` — what the mobile UI links to via "Voir la commande" |
| `message` | string | Plain French, shown as-is on the monitor screen (e.g. "Le stock n'a pas pu être réservé") — never a raw error/exception string, matching the rest of the app's error-copy convention |
| `createdAt` | string (ISO) | |
| `resolvedAt` | string, optional | Set once a retry succeeds |
| `retryCount` | number | |

Read: `isAdmin()` only (matches `producerInvoices`'s own precedent — no
poste needs this). Write: the Worker's own system identity only, same
`isMombongoWebhook()`-style narrow custom-claim pattern already established
for Mombongo — a dedicated `automationWorker: true` claim on a bootstrap
service account, never `isAdmin()` directly, so a compromised admin session
can't forge a fake "success" row.

### Reservation — a field on `orders`, not a new collection

Proposed: `orders/{id}.stockReservation: { status: "reserved" | "failed" |
"released", items: [{ productId, quantity }], attemptedAt }`. Keeping it
on the order doc (rather than a separate `reservations` collection) means
"is this order's stock accounted for" is answerable from one doc read,
matching how `orders.payment` is already a nested map rather than its own
collection.

**Open question:** what does "reserve" actually decrement? There's no
`products.stock` field today (`roadmap.md #12` — storefront inventory has
been a known, unstarted gap). Reservation logic needs to read `stockPF`'s
running balance per format, which means this automation depends on the
production-stock automation already existing and being trustworthy — build
order matters, this can't ship first.

## Next lifecycle: order → reservation → fulfilment (documented, 2026-09; not implemented)

Corrects and supersedes steps 2-4 of "The Cloudflare Worker cron job"
below, whose original sketch wrote a `stockPF` Sortie row at *reservation*
time. Nothing today implements either version — no Sales/reservations
batch has shipped yet — so there is no migration to do, only a clearer
target to build against next time this area is picked up:

1. **Order confirmation creates a reservation, never a stock Sortie.**
   Confirming an order (`orders.payment.status` reaching `"completed"`)
   only ever writes `orders/{id}.stockReservation` (see "Reservation — a
   field on `orders`" above) — no `stockPF` row of any kind is written at
   this point. The physical, on-hand `stockPF` balance is untouched.
2. **A reservation reduces *available* quantity, never *on-hand*
   quantity.** "On-hand" is what `stockPF`'s own Entrée/Sortie ledger sums
   to — the actual physical bottle count. "Available" is on-hand minus
   every order's currently-`"reserved"` `stockReservation` quantity for
   that format — a derived, read-time figure, never its own ledger row.
   Every sales-facing screen must show *available*, never raw on-hand,
   once reservations exist — showing on-hand would let two orders both
   claim the same physical bottles.
3. **Cancellation releases the reservation.** An order moving to
   `"cancelled"` with a prior `"reserved"` `stockReservation` flips it to
   `"released"` — still no `stockPF` write, since none was ever made for
   it. Available quantity goes back up simply because the reservation no
   longer counts against it; there is nothing to reverse in the ledger.
4. **Fulfilment closes the reservation and creates exactly one immutable
   Sortie.** Only when an order is actually fulfilled (delivered) does a
   real `stockPF` Sortie row get written — one per format, a deterministic
   id in the same spirit as the Entrée side (never `Crypto.randomUUID()`),
   immutable once created (same create-only rule shape). This is the one
   moment the on-hand balance actually decreases.
5. **Fulfilment creates one linked vente.** The existing `VTE-ORD-*`
   deterministic-id convention (step 4 of the cron job below) is the vente
   this Sortie is linked to — one vente per fulfilled order, never one per
   line item and never one per retry.
6. **Retries can never deduct twice.** The Sortie write and the vente
   write both use deterministic ids derived from the order (and format) —
   a retried/replayed fulfilment attempt always targets the exact same
   documents, and Firestore's create-only rule (no `update`/`delete`)
   makes a second attempt a no-op. This is not a new mechanism — it's the
   identical idempotency shape the Entrée side already proves out (see
   "Idempotency, including the lost-ack case" above), applied to the
   Sortie side once it exists.

## The Cloudflare Worker cron job

New file, `AROM-Production/src/routes/api/automations/run.ts` (or a Nitro
scheduled-task handler — exact wiring TBD against Nitro's `cloudflare-module`
preset, confirmed feasible via `wrangler.jsonc`'s `triggers.crons`, not yet
implemented). Every run, in order (each step only depends on the previous
one's writes, never re-reads its own output mid-run) — **steps 2-3's
Sortie-at-reservation-time sketch is superseded by "Next lifecycle" above;
read that section first**:

1. Query `qualityControls` newly reaching `decision: "liberer"` since the
   last successful run (needs a `processedForStockAt` marker field on
   `qualityControls` to avoid reprocessing — same idempotency concern
   `receptionSync.ts` already solved client-side, applied server-side
   here), join back to the `productions/{productionId}` doc for the
   bottle-format quantities → write `stockPF` Entrée rows (one per format)
   + `stockMP` Sortie rows for the raw material consumed → log an
   `automationRuns` row per lot released. A `productions` doc with no
   quality control yet, or one still in `quarantaine`/`rejeter`, never
   reaches this step — matches decision #3 above.
2. Query `orders` where `payment.status` is newly `"completed"` and
   `stockReservation` is unset → attempt reservation against `stockPF` →
   write `stockPF` Sortie rows (or Ajustement) + `orders.stockReservation`
   + an `automationRuns` row, `"needs-review"` if insufficient stock.
3. Query `orders` newly `"cancelled"` with a prior `"reserved"`
   `stockReservation` → write `stockPF` Entrée rows to release it → log.
4. (Logged only, no write) Query `ventes` docs whose `numero` matches the
   `VTE-ORD-*` deterministic-id pattern created since the last run → one
   `automationRuns` row per delivery-triggered sale, `type:
   "delivery-sale"`, `status: "success"` always (it already succeeded by
   the time this job sees it) — purely for the monitor screen's visibility
   promise, per decision #2 above.

## Retry ("Réessayer" on the monitor screen)

A new admin-only HTTPS route, `POST /api/automations/retry`, mirroring
`verifyMombongoCaller.ts`'s existing pattern exactly (verify the caller's
Firebase ID token, require `role === "admin"`) — re-runs step 2's
reservation logic for exactly the one `automationRuns` doc id given,
updating it to `success`/`resolvedAt` or leaving it `needs-review` again
with an incremented `retryCount`.

## Settings storage (`Paramètres opérationnels` / `Réglages de production`)

Extend `config/parametres` rather than create a new doc — this is the same
"one campaign-wide settings singleton" concept the dashboard's own
"Paramètres ERP" section already edits. New fields:
`rendementAttenduLPour100Kg`, `ecartAcceptablePct` (5 \| 10 \| 15). Per
decision #4 above, "Perte habituelle" is not a new field — it reads/writes
the existing `tauxPertesMax`. Bottle formats/prices (`Conditionnement`,
`Prix` rows in the first mockup) already exist as `prix500/330/300` +
`products` — no new field needed there, just a mobile settings UI
reading/writing what's already real.

## Firestore rules additions needed

- `stockPF/{id}` — **shipped, for the `"Entrée"`/`"production"` shape only**
  (see the `stockPF/{id}` section above for the actual, corrected rule —
  `isAdmin() || isProductionStaff()` create, no `isUnscopedStaff()`; this
  bullet's original "same shape as `stockMP`" sketch predates that and is
  superseded). Still open: the `"Sortie"`/`"Ajustement"` shapes for
  reservation-release/sale-deduction, whose write predicate is a future
  batch's decision, not yet made.
- `automationRuns/{id}` — admin read-only; write restricted to the new
  `automationWorker` custom claim (provisioned the same one-time,
  offline way `provision-mombongo-webhook-account.mjs` did for Mombongo).
- `orders/{id}.stockReservation` — needs `isValidInvoiceTransition`-style
  transition validation so a client can never forge a `"reserved"` status
  directly; only the Worker identity and the retry route's own caller
  identity (still the Worker, since retry re-runs server-side) may write
  it.

## What's still genuinely open

1. **Reservation failure UX**: the mockup's "Connexion nécessaire" pill on
   CMD-1048 implies some reservation failures are about a stale/expired
   session rather than genuinely insufficient stock. Default until told
   otherwise: mockup flavor text, not a real distinct failure mode —
   `automationRuns.message` stays one plain-French string, no sub-reason
   enum. Cheap to add later if a real second failure mode shows up.
2. **Polling interval**: proposed at 2 minutes, not yet load-tested against
   real Worker invocation costs. Default until told otherwise: ship at 2
   minutes, revisit only if it turns out to matter in practice.
3. **Build order (proposed, not yet confirmed)**: production-stock (①,
   now QC-gated per decision #3) must ship and be trusted before
   reservation (②/③) can mean anything real. `Paramètres opérationnels` +
   `Réglages de production` need no automation dependency at all — they're
   a plain settings UI over `config/parametres` and could ship
   independently of everything else in this document. `Automatisations`
   depends on ① and ② both existing and being trusted with real data.
