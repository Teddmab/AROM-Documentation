# Mombongo integration — deep audit (2026-09-17)

Read-only investigation across `AROM-Mobile`, `AROM-Production`, `AROM-Backend`,
and live infrastructure (Firestore, the deployed Cloudflare Worker, and
Mombongo's own API, probed directly). Every claim below is backed by a
file:line citation or a live command run during this audit — nothing here is
inferred from doc comments alone.

## Direct answer

**Is an invoice created on Mombongo currently reflected into the app? No —
and it currently *cannot* be, for reasons on both sides:**

1. **Mombongo's own API is currently down.** The exact endpoints AROM depends
   on (`createExternalInvoice`, `getExternalPublishedListings`) return HTTP
   500/503 "service not available" right now (verified live, see Evidence
   log). This is outside AROM's control.
2. **Even when Mombongo's side works, the app doesn't show a new invoice
   automatically.** There is no live listener for a *new* invoice appearing —
   only a once-known invoice's own payment status is watched live, and only
   during active checkout. A new invoice surfaces only after an admin
   re-opens/foregrounds the Admin Home screen (throttled to once/60s).
3. **The backend code that would make this work has never been committed to
   git**, and the live Worker that hosts the webhook handler has not been
   redeployed in 16 days — so even a code fix today doesn't reach production
   without a deliberate commit + deploy.

The code itself, where it exists, is well-built (idempotent webhook,
signature verification, sensible fallback matching). The blockers are
external (Mombongo's outage) and operational (nothing committed, nothing
redeployed), not a design flaw in what's written.

---

## 1. Invoice reflection: Mombongo → app

### 1.1 Architecture (as designed, and as it exists in code)

```
Mombongo (europe-west1-mombongo-dev.cloudfunctions.net)
   │  POST, HMAC-signed, x-mombongo-signature header
   ▼
AROM-Production  /api/webhooks/mombongo   (src/routes/api/webhooks/mombongo.ts)
   │  verifies signature against externalIntegrations/mombongo.outboundVerifySecret
   │  event="payment_complete"  → producerInvoices/harvestInvoices.statut = "payee"
   │  event="invoice_issued"    → creates harvestInvoices/{mombongoInvoiceId}, doc id = Mombongo's own id
   ▼
Firestore: producerInvoices / harvestInvoices (arom-production-657f2)
   ▲
   │  pull-sync, ADMIN role only, capped getDocs(max 20), NOT onSnapshot
AROM-Mobile  src/features/sync/syncPlan.ts:267-280
   ▼
AsyncStorage cache  →  producer-invoices.tsx ("Factures" screen)
```

### 1.2 What actually happens when a new invoice appears

- The **only** places in the entire mobile app with a live Firestore
  listener are `src/lib/firebase/auth.tsx:77` (the user's own profile) and
  `src/app/producer-invoice/[id]/pay.tsx:129` — and that second one only
  watches **one already-known invoice's own document**, only while the user
  is actively on the "waiting for payment" step of checkout.
- The `harvest-invoice/[id]/pay.tsx` screen's own copy claims *"Le statut
  sera mis à jour automatiquement dès que Mombongo confirme"* (status
  updates automatically) but **has zero `onSnapshot`/Firestore calls at
  all** — that promise is currently false for the harvest-invoice flow.
- A brand-new invoice (whether AROM-submitted or Mombongo-originated via
  `invoice_issued`) only enters the app's cache when `refreshRolePlan`
  re-runs — which only happens via `useAutoRefresh`, wired up **only** on
  `AdminOverviewScreen` (foreground + focus, throttled to once/60s). The
  dedicated "Factures" list screen (`producer-invoices.tsx`) has no
  auto-refresh of its own; pull-to-refresh there re-reads the *cache*, not
  Firestore.
- **Net effect**: even with a perfectly working Mombongo backend, an admin
  who never opens/foregrounds the Home screen will not see a new invoice
  until they do — there is no push/live path to the invoice list itself.

### 1.3 Backend correctness (the webhook handler itself)

`AROM-Production/src/routes/api/webhooks/mombongo.ts` is genuinely solid:
signature verification before parsing, idempotent (repeat delivery is a
200 no-op via a terminal-status check), sensible three-tier invoice
matching (direct id → harvest id → legacy `mombongoInvoiceId` field
fallback), fails closed on bad signature (401), acks-but-logs on an
unmatched invoice id rather than looping Mombongo's retries forever.

### 1.4 Live-infrastructure findings (verified during this audit, not assumed)

- **`externalIntegrations/mombongo` exists in the live Firestore project**
  (`arom-production-657f2`), `active: true`, `baseUrl` =
  `https://europe-west1-mombongo-dev.cloudfunctions.net` — Mombongo's
  **dev/staging** environment, not necessarily a production one.
- The **`mombongo-webhook@system.arom.cd`** system account exists, is not
  disabled, and carries the correct `mombongoWebhook: true` custom claim.
- **Two `producerInvoices` documents exist for real**, both clearly
  test/verification records (`arom-e2e-…`, `arom-verify-…`, ids carrying
  Unix-ms timestamps that decode to 2026-08-31/09-01). One of them has
  `statut: "payee"` — **proof the inbound webhook did work end-to-end at
  least once**, on 2026-09-01.
- **`harvestInvoices` and `harvestOffers` are both completely empty (0
  documents)** — the "Sprint DP" marketplace-offer flow has never produced
  a real record.
- **The live Worker (`arom-production.purple-hat-3cb3.workers.dev`) was
  last deployed 2026-09-01 07:37 UTC — 16 days ago.** Probed directly:
  `/api/webhooks/mombongo` and `/api/mombongo/create-invoice` both respond
  401 (route exists, signature/auth check runs) — so the Mombongo routes
  *are* live. But `/api/inventory/confirm-order` (this session's Sprint 08
  work, committed to `main` on 2026-09-16) returns **404** — the live
  Worker predates that work entirely. **Nothing committed after
  2026-09-01 is live**, Mombongo-related or not.
- **Mombongo's own API is currently broken.** Direct probes today:
  - `POST /createExternalInvoice` → HTTP 503 "The service you requested is
    not available yet"
  - `POST /getExternalPublishedListings` → HTTP 500 "The server encountered
    an error"
  - A wrong/nonexistent path on the same host → clean 404 "Page not found"
    (confirms these two functions are registered but broken, not simply
    absent).
  - `AROM-Backend/scripts/check-mombongo-deployment.mjs` (a pre-existing
    smoke-test script) was run live during this audit and failed all 3 of
    its checks — its own doc comment says it passed cleanly on 2026-08-31.
    **This is a regression on Mombongo's side since then**, not a change
    on AROM's side.

### 1.5 Payment-gateway reachability in a real build

`getPaymentGateway.ts` picks the real Mombongo gateway only when
`EXPO_PUBLIC_ENABLE_MOMBONGO_SIMULATION=true` (a legacy flag name — it now
gates the *real* gateway, not a simulation). Checking every `eas.json`
profile:

| Profile | Flag set? | Result |
|---|---|---|
| `preview` | yes | real gateway reachable |
| `qa-apk` | no | `unavailableGateway` — "Payer la facture" is disabled |
| `production` | no | `unavailableGateway` — same |

**No production or QA build of the app can currently even attempt to call
Mombongo** — only ad hoc internal `preview` builds can. This is presumably
intentional caution rather than an oversight, but it means "does the real
app work" and "does the `preview` test build work" are different
questions today.

---

## 2. "Product from Mombongo": the marketplace browse/offer flow

This is a **separate, distinct flow** from invoices — worth being precise
about, since the two are easy to conflate:

- `producerInvoices`/`harvestInvoices` = AROM **paying** a producer/farmer
  (AROM is the payer).
- Harvest listings/offers (Sprint DP) = AROM **buying raw commodity** on
  Mombongo's own farmer marketplace (AROM is the buyer, browsing what
  farmers are selling).

**Mombongo does not populate AROM's own product catalog** (`products`) or
storefront `orders` — this is documented explicitly in
`data-model.md`: *"Mombongo does not write to `products` or `orders`... If
AROM's supply/product data is meant to sync from Mombongo, that is a
distinct, unbuilt capability."* If "product from Mombongo appearing here"
was expected to mean AROM's own sales catalog, **that doesn't exist
anywhere in the codebase** — confirmed by grep across all three repos.

### 2.1 What does exist: harvest listings

`src/app/harvest-listings.tsx` (AROM-Mobile) is a **live, uncached**
screen — deliberately never synced through the offline cache (own doc
comment: a stale cached listing risks offering on something no longer
active). It calls `POST /api/mombongo/listings` on mount, on
pull-to-refresh, and per search, which in turn calls Mombongo's real
`getExternalPublishedListings` — currently returning 500 (see §1.4).
Submitting an offer (`harvest-listing/[id]/offer.tsx`) is the same
pattern: a live call to `/api/mombongo/create-offer`, no Firestore write
from mobile itself.

**Expected listing shape** (`HarvestListing`,
`AROM-Mobile/src/features/harvest/mombongoHarvestGateway.ts:32-41`):
```ts
{ id, commodity, province, territory, quantityKg, quality: "A"|"B"|"C", pricePerKgCdf, sellerId }
```

Since `harvestOffers` is empty in the live Firestore, **this flow has
never produced a real accepted offer** — consistent with the listings
endpoint currently being broken.

---

## 3. What AROM expects from Mombongo's API — full contract as implemented

All calls are `POST`, `x-partner-id` + `x-partner-signature` (HMAC,
`mombongoSigning.ts`) headers, base URL from `externalIntegrations/mombongo.baseUrl`.

| Endpoint | Called by | Purpose | Expected response shape |
|---|---|---|---|
| `POST /createExternalInvoice` | `mombongo.ts:96` | AROM asks Mombongo to create a payable invoice for a `producerInvoices` doc | `{ status: "accepted"\|"duplicate_ignored", invoiceId }` |
| `POST /createExternalInvoiceCheckout` | `mombongo.ts:143`, `mombongoHarvest.ts:146` | Start a checkout session (card or mobile money) for an already-created invoice | `{ status: "checkout_created", providerRef, clientSecret?, depositStatus? }`; `409` = already in progress, `404` = not found, `502` = provider rejected |
| `POST /getExternalPublishedListings` | `mombongoHarvest.ts:38` | Browse farmer harvest listings | `{ listings: HarvestListing[] }` |
| `POST /createExternalHarvestOffer` | `mombongoHarvest.ts:65` | Submit an offer on a listing | `{ status: "accepted", offerId }`; `400` = rejected (inactive listing/bad qty/price) |
| *(inbound, Mombongo → AROM)* `POST /api/webhooks/mombongo` | Mombongo calls AROM | Two events on one URL | `event: "payment_complete"` → `{ externalInvoiceId, status }`; `event: "invoice_issued"` → `{ invoiceId, farmerId, listingId, amountUsd, quantityKg, commodity }` |

Notable gaps in what AROM can currently learn from Mombongo:
- **No polling/status endpoint** — per Mombongo's own spec (quoted in
  `mombongoHarvest.ts:91`), "poll is not available yet" for offer
  won/declined; AROM only learns "won" via the `invoice_issued` webhook
  matching a `listingId` back to a pending offer. A declined offer is
  indistinguishable from "still pending" — the code is honest about this
  rather than guessing.
- **`testMode` is hardcoded `true`** in the checkout write
  (`mombongo.ts:194`, `mombongoHarvest.ts:182`) with a comment flagging it
  should be revisited once a real response is seen — Mombongo's contract
  doesn't echo this field on the checkout response itself.
- **Currency**: Mombongo's contract is USD-only; AROM converts FC/CDF via
  a locally-cached exchange rate (`config/exchangeRate`, seeded from
  Mombongo's own `config/exchange_rate.usdToCdf`, per `mombongo.ts:72-76`).

---

## 4. Deployment and commit hygiene — the operational blockers

- **Zero commits, ever, anywhere in `AROM-Production`'s git history touch
  anything Mombongo-related** (`git log --all --grep=mombongo` and
  `git log --all -- '*mombongo*'` both return nothing). Every file listed
  in §1/§2/§3 above is currently **uncommitted, untracked working-tree
  content**, last edited 2026-08-27 through 2026-09-01. If this machine's
  working tree were ever lost or reset, this entire integration — despite
  being real, tested-once, working code — would disappear with no git
  history to recover it from.
- The live Worker's last real deployment was **2026-09-01 07:37 UTC**.
  Anything committed or edited since then, Mombongo-related or not
  (including this session's entire Sprint 08 stock/order-reservation
  work), is not live.
- `AROM-Production/src/lib/payments/pawapay.ts` also exists, uncommitted,
  dated 2026-08-14 — a second, apparently-earlier payment-provider
  integration (PawaPay) not covered by this audit; worth a follow-up look
  if it's still intended to be used.
- Code-comment staleness: several files (`getPaymentGateway.ts`'s own
  neighbors — `gateway.ts`, `unavailableGateway.ts`,
  `invoices/model.ts:47,49`) still describe the Mombongo gateway as
  "not yet built" / simulation-only, while the gateway file itself and
  its test both describe it as live since 2026-09-01. The comments were
  never reconciled after the real gateway was added.

---

## 5. Evidence log (commands run live during this audit)

```
# Live Worker route probes (arom-production.purple-hat-3cb3.workers.dev)
POST /api/webhooks/mombongo         → 401 (route live, signature check runs)
POST /api/mombongo/create-invoice   → 401 (route live, auth check runs)
POST /api/mombongo/listings         → 401 (route live, auth check runs)
POST /api/inventory/confirm-order   → 404 (this session's Sprint 08 work — NOT deployed)

# Mombongo's own API (europe-west1-mombongo-dev.cloudfunctions.net)
POST /createExternalInvoice           → 503 "service not available yet"
POST /getExternalPublishedListings    → 500 "server encountered an error"
GET  /createExternalInvoice           → 500 (not even a clean 405)
GET  / (root, wrong path)             → 404 clean (confirms above are real, registered, broken functions)

# scripts/check-mombongo-deployment.mjs (pre-existing script, run live)
3 of 3 checks FAILED — doc comment says all passed on 2026-08-31

# Live Firestore (arom-production-657f2), read-only
externalIntegrations/mombongo: exists, active=true, all secret fields set
mombongo-webhook@system.arom.cd: exists, not disabled, custom claim { mombongoWebhook: true }
producerInvoices: 2 docs — arom-e2e-1788248783561 (statut=payee), arom-verify-1788185574920 (statut=approuvee)
harvestInvoices: 0 docs
harvestOffers: 0 docs

# git history (AROM-Production, all branches)
git log --all --oneline -i --grep="mombongo"          → (empty)
git log --all --oneline -- "*mombongo*" "*Mombongo*"  → (empty)

# Worker deployment history
Last deployment: 2026-09-01T07:37:11.756Z (16 days before this audit)
```

---

## 6. Open questions worth putting to Mombongo / deciding internally

1. Is `europe-west1-mombongo-dev.cloudfunctions.net` meant to be a
   long-lived integration target, or a temporary dev sandbox that was
   expected to be replaced by a real partner-production URL? The `-dev`
   in the hostname and its current 500/503 state both suggest the latter.
2. Has AROM actually been provisioned as a partner on Mombongo's
   production side (not just dev)? `check-mombongo-deployment.mjs`'s own
   comment says as of 2026-08-31 it had not been.
3. Who owns re-deploying `AROM-Production`'s Worker going forward, now
   that Lovable's auto-deploy pipeline is gone? Nothing has shipped in 16
   days.
4. Should the uncommitted Mombongo code be committed as-is (it's real,
   working code, just never checked in), reviewed first, or reworked
   given the doc-comment staleness noted in §4?
5. Does the harvest-invoice pay screen's "updates automatically" claim
   need a real listener added, or should the copy be corrected to match
   the producer-invoice screen's honest "check back" framing until one
   exists?
