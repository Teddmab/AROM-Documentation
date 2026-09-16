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
margins, the automation monitor screen, a generic inventory engine. Design
for reservation/sale-deduction/cancellation-release is now approved — see
"Reservation and balance projections" below — but still unbuilt as of this
writing; expiry and forecasts/margins remain genuinely out of scope. The
Worker cron job below remains a *future reconciliation* mechanism only,
now confirmed **not** the reservation mechanism itself either (see that
section) — nothing in this batch depends on it running, and normal stock
visibility never waits for its poll interval.

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

### Reservation and balance projections — approved architecture (Sprint 08, 2026-09; not yet implemented)

Supersedes the single-field `orders.stockReservation` sketch above and the
Worker-cron-driven timing in "The Cloudflare Worker cron job" below.
Approved after a read-only audit + design pass (see the Sprint 08 decision
report for the full audit trail); nothing below is built yet.

**Why not just sum `stockPF` live.** A Firestore transaction can `get()` a
bounded set of documents by reference — with real optimistic-concurrency
protection — but it cannot run a consistent, contention-safe aggregation
query inside that same transaction. Two concurrent reservation attempts
could each query-sum "5 available," and both succeed, overselling. A
maintained balance *document*, read and written by reference inside the
transaction, is required — not an optimization, a correctness requirement.

#### Two balance projections, both fed by the same authoritative event

**`stockBalance/{format}`** — exactly 3 documents (`500ml`/`330ml`/`300ml`,
`stockPF`'s own no-space convention — see "Format canonicalization"
below), the single global contention point:

| Field | Type | Notes |
|---|---|---|
| `format` | `"500ml"` \| `"330ml"` \| `"300ml"` | Matches the doc ID |
| `onHand` | number | Mirrors `stockPF`'s own Entrée−Sortie balance for this format |
| `reserved` | number | Sum of every not-yet-released/fulfilled reservation's quantity for this format |
| `updatedAt` | string (ISO) | |

**`stockLotBalance/{productionId}_{format}`** — one document per
(released production lot, format) combination, preserving end-to-end lot
traceability without requiring the order/storefront layer to ever know a
lot exists:

| Field | Type | Notes |
|---|---|---|
| `productionId` | string | |
| `qualityControlId` | string | The releasing control — same reference `stockPF`'s own Entrée row for this lot carries |
| `lot` | string | Denormalized label, same convenience `qualityControls.lot` already provides elsewhere |
| `format` | `"500ml"` \| `"330ml"` \| `"300ml"` | |
| `onHand` | number | |
| `reserved` | number | |
| `releasedAt` | string (ISO) | The releasing control's own `date` |
| `updatedAt` | string (ISO) | |

Neither document stores a calculated `available` field — it is always
`onHand − reserved`, computed at read time, so it can never drift from its
own inputs by construction.

**Both projections are written in the SAME transaction as the existing,
already-shipped `stockPF` Entrée write** (`AROM-Mobile`'s
`qualitySync.ts`'s `syncOne`, extended, not replaced): a QC release
increments `stockBalance/{format}.onHand` and creates/increments
`stockLotBalance/{productionId}_{format}.onHand` for every format present,
alongside the Entrée row it already writes today. One authoritative event,
three consistent effects, one transaction.

#### FIFO lot allocation

A reservation (order confirm) or a direct sale allocates across
`stockLotBalance` documents automatically, never asking the customer or
partner to choose a lot:

1. Order by the releasing control's `releasedAt` timestamp, oldest first.
2. Deterministic tie-breaker on exact-equal timestamps: `productionId`
   (equivalently, the `stockLotBalance` doc ID) ascending — never
   insertion order, never "whichever the query happened to return first."
3. Skip a lot whose `available` (`onHand − reserved`) is zero.
4. A single format's requested quantity may span multiple lots.
5. A lot with no released, terminal head control (unreleased, still in
   quarantine, or rejected) never has a `stockLotBalance` document at
   all — nothing to skip, it was never eligible to begin with.

The resulting allocation is recorded exactly, not re-derived later:

```
orders/{id}.reservation = {
  status: "reserved" | "released" | "fulfilled",
  items: [
    {
      format,
      quantity,
      allocations: [
        { productionId, qualityControlId, quantity }
      ]
    }
  ],
  reservedAt,
  reservedByUid
}
```

`orders.items` (the existing, customer-facing line items) is unchanged —
`reservation` is an additive field, same shape-of-change as `payment`
before it. Lot selection never appears in any order-facing UI, mobile or
web; it is an internal allocation detail recorded for traceability and
consumed verbatim at fulfilment (so fulfilment decrements exactly the lots
that were actually reserved, never re-running FIFO a second time).

#### Trusted write boundary — reused Worker pattern, not client writes, not Cloud Functions

**Rejected: client-authored Firestore writes validated by Rules alone**
(the originally-proposed Tier 1/Tier 2 rules strategy). Proving `reserved`
increments by exactly the right amount, choosing a FIFO allocation, and
validating prices/totals against live `products` data are all real
business logic a Rules expression is the wrong tool to reimplement, and a
Rules-only design would have to either trust client-computed totals
(exactly what the audit warned against) or hand-write brittle
per-array-sum expressions Firestore's rules language doesn't cleanly
support.

**Approved: reuse the existing `/api/mombongo/*` trusted-server-route
pattern**, already shipped and running in the same Cloudflare Worker this
app deploys to — confirmed capable during the Sprint 08 audit, not merely
assumed:

- **Token verification without the Admin SDK**: `verifyFirebaseIdToken.ts`
  calls Google's own `accounts:lookup` REST endpoint directly over plain
  `fetch` — the Admin SDK doesn't run in this Worker (gRPC/Node-internals
  dependency), so this is the real, already-proven substitute, not a new
  risk.
- **Transactions without the streaming SDK**: `firebase/firestore/lite`
  (via `serverDb.ts`) — REST-based, one-shot calls, no `onSnapshot`
  (which genuinely hangs in this Worker runtime) — **does** export
  `runTransaction`/`writeBatch` with the identical contract and guarantees
  the full SDK documents (Firestore transactions are a server-coordinated
  REST protocol; only real-time listening needs a persistent stream).
  Verified directly against the installed package's own type declarations
  during this audit, not assumed from the full SDK's behavior.
- **A dedicated, narrow system identity** signs in and performs the
  transaction — mirroring `mombongoSystemAuth.ts`'s
  `signInAsMombongoSystem()` exactly, but its own separate identity
  (`inventoryService: true` custom claim, its own provisioning script,
  its own env-var-sourced credentials) — never reusing the Mombongo
  identity for an unrelated domain, same "narrow, single-purpose"
  principle that identity was built on.

New TanStack Start server routes (`src/routes/api/inventory/*`), each
following `create-invoice.ts`'s exact shape (`createFileRoute` +
`server.handlers.POST`):

- `confirm-order` — pending → confirmed, reserve.
- `cancel-order` — confirmed → cancelled, release.
- `fulfil-order` — confirmed → fulfilled, consume reservation, Sortie, vente.
- `direct-sale` — a manual (no-order) sale, same trusted deduction path.

Each handler: verifies the caller's ID token → loads their `users/{uid}`
profile (via the system identity, mirroring `verifyMombongoCaller.ts`) →
checks `active == true` and an authorized poste/role for the specific
action → loads the current `orders`/`products` state itself (never trusts
client-sent totals/formats/prices/allocations) → canonicalizes formats →
runs one `runTransaction` doing the real work → returns a plain-language
result (success, or a structured insufficient-stock/error response) →
never leaves a partial write on any failure path (a transaction either
fully commits or writes nothing).

#### Authorization

| Action | Allowed | Denied |
|---|---|---|
| Order confirm/cancel/fulfil, direct sale | `isAdmin()`, `isCommercialStaff()` (Chargée de Commercialisation / mobile SALES) | Personnalisé/unscoped, missing/unknown poste, Agent de collecte, a partner calling this endpoint directly, inactive accounts |
| QC release (existing, extended to also write both balance projections) | `isAdmin()`, `isProductionStaff()` (unchanged from the authorization-audit batch) | Same denial set as `stockPF` create today |

Partners keep their existing, unchanged path: creating their own `pending`
order, and cancelling their own still-`pending` order — neither of those
touches a reservation (nothing is reserved until staff confirm), so
nothing here narrows what a partner could already do.

#### Order confirmation (pending → confirmed)

1. Aggregate the order's `items` by canonical format.
2. Validate every `productId`/`format` against live `products` data —
   never trust the order's own snapshot for this decision, only for
   display.
3. Inside one transaction: `tx.get()` every relevant `stockBalance/{format}`
   and candidate `stockLotBalance/*` document.
4. If `available < requested` for **any** format, abort the whole
   transaction — the order stays `pending`, nothing is reserved anywhere,
   and the response names the exact requested/available/missing quantity
   per affected format (see "UX" below).
5. Otherwise: allocate FIFO per format, increment `stockBalance.reserved`
   and every selected `stockLotBalance.reserved`, write
   `orders/{id}.reservation` (status `"reserved"`, exact allocations,
   `reservedByUid`), set `orders.status = "confirmed"`.

`onHand` never changes at this step, on any document.

#### Order cancellation (confirmed → cancelled)

Requires `reservation.status == "reserved"` (read inside the same
transaction, before any write — a retry that finds it already
`"released"` no-ops rather than double-releasing). Decrements
`stockBalance.reserved` and every allocated `stockLotBalance.reserved` by
the recorded allocation, sets `reservation.status = "released"`,
`orders.status = "cancelled"`. `onHand` never changes.

#### Order fulfilment (confirmed → fulfilled)

Requires `reservation.status == "reserved"`. Re-verifies the recorded
allocation against current `stockBalance`/`stockLotBalance` reserved
quantities (defensive — should always match, since nothing else can alter
a `"reserved"` reservation), then in one transaction: decrements
`stockBalance.reserved` **and** `.onHand`, and each allocated
`stockLotBalance.reserved` **and** `.onHand`; writes one immutable
`stockPF` Sortie row per (order × production lot × format) combination
present in the allocation, deterministic id
`PF-OUT-{orderId}-{productionId}-{format}`; writes the linked `ventes` row(s)
via the existing `VTE-ORD-{orderId}-{idx}` deterministic bridge (unchanged
shape); sets `reservation.status = "fulfilled"`, `orders.fulfilledAt`,
`orders.status = "fulfilled"`.

Stock is deducted **exactly once**, at fulfilment — never also at
confirmation.

#### Direct sales (no order)

Both apps' existing manual-sale paths (`AROM-Production`'s
`CommercialisationSection`, `AROM-Mobile`'s `sale-new`) are re-pointed at
the same `direct-sale` trusted endpoint rather than writing `ventes`
directly. FIFO-allocates automatically unless the staff member's own
already-selected `productionIds` remain valid (both apps already collect
this as a manual field — honored when still available, not overridden),
validates availability first, and on success: writes the Sortie row(s),
decrements both balance projections' `onHand` (never `reserved` — a
direct sale has no reservation phase), writes the `vente` exactly once.
On insufficient stock: the draft is preserved client-side, affected
formats are identified, quantities can be adjusted — no partial sale, no
partial stock effect, ever.

No direct-sale path may write `stockPF`/`ventes` directly once this ships
— that would silently bypass the same availability check the order path
enforces.

#### Idempotency

Every handler reads the order/reservation's current state first and gates
on it, exactly the lost-ack-safe shape already proven for QC release
(`qualitySync.ts`'s `syncOne`): a retried confirm/cancel/fulfil call that
finds the target state already reached no-ops rather than re-applying a
delta; a genuine mismatch (recorded allocation disagrees with current
balance state) raises a real, reported conflict rather than silently
re-deriving one.

#### Format canonicalization

One shared, documented mapping — not scattered string replacement:
`"500 ml" ↔ "500ml"`, `"330 ml" ↔ "330ml"`, `"300 ml" ↔ "300ml"`. An
unrecognized format on either side is rejected, never coerced to a
default. Lives in one module both apps' inventory-touching code imports
(mirrors how `STOCKPF_FORMAT_BY_BOTTLE_KEY` already centralizes the
`q500`/`q330`/`q300` ↔ `"500ml"` mapping on the receipt side — this is
its sibling for the with-space/no-space product-facing convention).

#### Not implemented, on purpose

Partial fulfilment (no per-item status, no partial-delivery field — an
order is fulfilled all at once or not at all, matching the schema and UI
as they exist today). Reservation expiry (no duration, no warning, no
payment-state interaction, no extension — a reservation lives until
explicit cancellation or fulfilment, full stop, until AROM asks for
otherwise and approves the specific parameters).

#### Step B (built, 2026-09) — production packaging immutability boundary audit

Step B (`AROM-Production`'s `applyQcReleaseReceipt`) derives the stockPF/
stockLotBalance/stockBalance quantities it writes from `productions/{id}`'s
own `q500`/`q330`/`q300` fields, read transactionally at the moment of
receipt. The preferred invariant is: packaging may be corrected freely
before a terminal quality control exists for a production, but once a
terminal control (`decision: "liberer"` or `"rejeter"`) exists, packaging
identity and quantities should no longer be changeable through any
supported application flow — otherwise a production's own record and the
immutable inventory movements already derived from it can silently
diverge.

**What's actually enforced today, and by what:**

- **Application-level (the real boundary today):** grepped every write
  path in both `AROM-Mobile` and `AROM-Production` (2026-09 hardening
  audit) — `productions/{id}` has exactly one writer anywhere in either
  app, `productionSync.ts`'s `syncOne`, and it is unconditionally
  create-only: `if (existing.exists()) return;` before any `tx.set`. No
  edit/correction screen for an existing production exists in either app.
  So today, packaging is never changed after a production document first
  syncs, let alone after a terminal control — this holds regardless of
  whether any QC exists yet at all.
- **Firestore Rules (not enforced today):** `productions/{id}`'s `update`
  rule (`firestore.rules`) currently allows any admin/production-staff
  identity to overwrite the whole document, unconditionally — Rules do
  not check whether a terminal `qualityControls` document exists for this
  `productionId`, because they structurally cannot cheaply: `qualityControls`
  documents are keyed by their own id, not by `productionId`, so "does a
  terminal control exist for this production" is a collection *query*
  (`where("productionId", "==", id)`), and Firestore Rules cannot execute
  queries or aggregates over a collection — only `get()`/`exists()` against
  a single, already-known document path. There is no such fixed path here
  (a production can have zero, one, or — before a control is chosen —
  transiently more than one candidate control racing to become the terminal
  one; see `qualitySync.ts`'s own candidate-query-then-transactional-reread
  pattern for why that can't be collapsed to one deterministic id either).

**Residual risk:** the Rules layer alone does not prevent a future code
path (a not-yet-built production-correction screen, or a compromised/
misused admin session) from editing `q500`/`q330`/`q300` on a production
that already has inventory movements derived from its current values.
Today this risk is theoretical, not reachable — no such edit path exists
in either shipped app. It becomes real the moment one is built.

**Deliberately not done in this pass** (would broaden this hardening
step into a production-correction system, which is out of scope): adding
a Rule that locks `q500`/`q330`/`q300` once first set (the simplest
Rules-provable proxy — it doesn't need to know whether a QC exists at
all, just whether the field was previously defined) or a denormalized
`productions/{id}.hasTerminalControl` flag set by the same transaction
that commits a terminal QC (closer to the preferred rule, but a new
cross-collection write `qualitySync.ts` doesn't make today). Either is a
reasonable follow-up; both require an explicit ADMIN decision on which
production-correction workflow (if any) should remain possible, which
this hardening pass does not make on its own.

## Next lifecycle: order → reservation → fulfilment (documented 2026-09; approved architecture above; not yet implemented)

The six principles below are unchanged in spirit from the original design
pass and now fully specified in the approved architecture above — kept
here as the short version:

1. **Order confirmation creates a reservation, never a stock Sortie.**
2. **A reservation reduces *available* quantity, never *on-hand*
   quantity.** Every sales-facing screen must show `onHand − reserved`
   (available), never raw `onHand`, once this ships.
3. **Cancellation releases the reservation**, on-hand untouched.
4. **Fulfilment closes the reservation and creates exactly one immutable
   Sortie per (order × lot × format)**, plus one linked vente per order —
   this is the one moment on-hand actually decreases.
5. **Fulfilment creates the linked vente(s) exactly once** — the existing
   `VTE-ORD-*` deterministic-id bridge, unchanged.
6. **Retries can never deduct twice** — deterministic ids everywhere, and
   every handler gates on the current state it reads before writing.

## The Cloudflare Worker cron job

**Superseded as the reservation mechanism (Sprint 08, 2026-09).** Order
confirm/cancel/fulfil and direct sales are now trusted, synchronous HTTP
endpoints (see "Trusted write boundary" above) — a cron poll was never
implemented and is no longer the plan for that path; a confirm action
needs its availability answer in the same request, not on the next poll
interval. What follows is now purely a *possible future reconciliation*
job (detect drift between the ledger and the projections, e.g. after a
manual Firestore console edit) — genuinely optional, nothing depends on
it, and it has never been built. Step 1 below describes what QC release
already does today (shipped, in `qualitySync.ts`), kept here only as
history of the original sketch.

1. ~~Query `qualityControls` newly reaching `decision: "liberer"`...~~
   Shipped a different way: the QC-release transaction itself writes the
   `stockPF` Entrée row (and, once Sprint 08 ships, both balance
   projections) synchronously, inside the same client transaction that
   creates the releasing control — never via a polled Worker job. No
   `processedForStockAt` marker field exists or is needed.
2. ~~Query `orders` where `payment.status` is newly `"completed"`...~~
   Superseded in full, not just in timing: reservation happens
   synchronously inside `confirm-order`'s own trusted transaction (see
   "Order confirmation" above), never on a poll interval, and is keyed off
   the `pending → confirmed` status transition, not `payment.status`.
3. ~~Query `orders` newly `"cancelled"`...~~ Superseded: `cancel-order`'s
   trusted transaction does this synchronously.
4. (Still potentially useful, unchanged in spirit) A read-only
   reconciliation pass — e.g. confirm `stockBalance.onHand` still equals
   the sum of that format's `stockPF` Entrée minus Sortie rows, and that
   `stockBalance.reserved` still equals the sum of every `"reserved"`
   order's allocation for that format — logged to `automationRuns` if it
   ever finds drift. Not required for Sprint 08's own correctness (every
   write path already keeps the projections consistent by construction);
   valuable only as a safety net against manual/out-of-band data edits.

## Retry ("Réessayer" on the monitor screen)

Superseded by Sprint 08's own idempotency model: since order confirm no
longer runs on a poll interval, there is no separate `automationRuns`
"needs-review" row to retry from a monitor screen — the confirm request
itself either succeeds or reports the exact insufficient-stock detail
synchronously (see "UX" in the Sprint 08 decision report), and the admin
retries by re-attempting the same action once stock changes, same as any
other trusted endpoint's error path elsewhere in this app
(`verifyMombongoCaller.ts`'s callers included).

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
- `automationRuns/{id}` — only relevant if the optional future
  reconciliation pass (see "The Cloudflare Worker cron job" above) is ever
  built; admin read-only, write restricted to its own dedicated custom
  claim. Not required for Sprint 08 itself.
- `stockBalance/{format}` / `stockLotBalance/{productionId}_{format}` /
  `orders.reservation` / `stockPF` Sortie rows — **no client-writable rule
  at all**, by design (Sprint 08 decision: rejected client-write Rules
  validation in favor of a trusted server route). `create`/`update` on
  each is restricted to the new `inventoryService` custom claim only
  (own provisioning script, own credentials — never reusing
  `mombongoWebhook`), mirroring `harvestOffers`/`harvestInvoices`' own
  "every write goes through the webhook identity, never a direct
  `isAdmin()` write" shape more than it mirrors `stockPF`'s own
  actor-authorized-client-write shape. `read` stays authorization-based
  (`isAdmin()`/`isProductionStaff()`/`isCommercialStaff()`, matching
  `stockPF`'s own read rule) since reading a balance is not a
  business-logic decision the way writing one is.

## What's still genuinely open

Sprint 08's own decision pass (2026-09) resolved every item this section
used to list (reservation failure UX, polling interval, Rules-vs-trusted-
route strategy, partial fulfilment, reservation expiry) — see the
"Reservation and balance projections" section above for each resolution.
What remains genuinely open, after that pass:

1. **Existing manual-sale UI flows still write `ventes` directly.**
   `AROM-Production`'s `CommercialisationSection` and `AROM-Mobile`'s
   `sale-new` need to be re-pointed at the new `direct-sale` trusted
   endpoint (Sprint 08 step E) — until that ships, those two paths remain
   capable of overselling even after the order path is protected. Not a
   design gap, a sequencing one: flagged so it isn't forgotten between
   steps C/D (order path) and E (direct sales).
2. **What an admin does with a migration-reported unreservable historical
   order.** The migration dry-run must report a confirmed order that
   can't be fully reserved (per the approved architecture, never force it
   negative) — but no resolution action is designed yet (manually adjust
   the order? partially reserve what's available and flag the rest?
   leave it unreserved and let fulfilment fail loudly instead?). Blocks
   nothing about steps A-E; blocks only actually running the real
   migration once the dry-run report comes back non-empty.
3. **Build order.** Step A (schemas/canonical formats/migration dry-run)
   before B (QC release writes both balance projections) before C/D
   (order reservation/fulfilment, meaningless without B already trusted)
   before E (direct sales) before F (UI). `Paramètres opérationnels` /
   `Réglages de production` remain independent of all of this, as before.
