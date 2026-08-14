# User & data flows

A role-by-role, page-by-page map of how AROM actually behaves, built by
tracing the code (not assuming it from the file names) — see
[architecture.md](architecture.md) for the system-level picture and
[rbac.md](rbac.md) for the role model itself. This doc exists because the
flow had drifted from what the pages actually did; keep it in sync with
`AROM-Production/src/routes/` as pages change.

## Role × page map (current, verified 2026-08-14)

| Role | Entry point | Reaches | Reads / writes | Verified gaps |
| --- | --- | --- | --- | --- |
| Visitor | `/` | `/login`, `/storefront/signup`, `/dashboard` (bounces to login) | none | — |
| Partner | `/storefront/signup` → `/storefront` | catalog, cart, own orders (live via `onSnapshot`) | `products` (read), `orders` (own — create, read; cancel while pending) | no product photos, no promo banner, no payment, order status has no delivery ETA |
| Staff | `/login` → `/dashboard` | sections listed in `menus` | full read/write on all internal collections — `firestore.rules` doesn't scope by `menus` (documented, deliberate for now, see roadmap #1) | account creation/deactivation is CLI-only |
| Admin | `/login` → `/dashboard` | everything | everything, incl. the order→`ventes` bridge | no catalog UI, no promo UI, no payment reconciliation UI |

Auth/session edge cases (deactivated account, no profile doc, already-signed-in
user hitting `/storefront/signup`) are now handled explicitly — see
[Teddmab/AROM-Production#5](https://github.com/Teddmab/AROM-Production/pull/5)
for what was broken and the fix.

## How data circulates today

```
Admin/staff → Firestore (direct writes, dashboard forms)
                 │
                 ├─ producteurs, approvisionnements, productions, stockMP,
                 │  clients, ventes, marketing, charges, config/parametres
                 │  (internal ERP — admin/staff only)
                 │
                 └─ products (catalog) ── read-only ──► Partner sees it on /storefront
                                                              │
                                                        places an order
                                                              │
                                                              ▼
                                                     orders (pending)
                                                              │
                                              admin confirms → fulfills
                                                              │
                                                              ▼
                                         ventes row(s) created (encaisse = 0)
```

Two things stand out as *disconnected* rather than *missing*:

- **`products`** is only ever written by `AROM-Backend/scripts/seed.mjs` or
  by hand in the Firestore console — there is no dashboard UI to add, edit,
  price, deactivate, or photograph a product, even though `storage.rules`
  already reserves `products/**` for images (roadmap #6, broadened below).
- **`clients` (internal registry) and `users` (partner accounts)** are two
  unlinked systems. A partner who orders through the storefront never gets
  a `clients` doc; a manually-entered `vente` has no link back to a real
  partner account. `ventes.idClient` is a free-text field, not a foreign
  key. Fine at current scale; worth knowing before it isn't.

## Target flow

```
Admin publishes/edits products (name, price, photo, active flag)
        │
        └─ optional promo banner (headline + product ref + dates)
                 │
                 ▼
Partner sees live catalog with photos + banner on /storefront
                 │
        chooses: pay by mobile money (PawaPay) OR pay on delivery
                 │
                 ▼
Order carries a real payment status, not just pending/confirmed/fulfilled
                 │
        admin confirms → fulfills
                 │
                 ▼
ventes row created with encaisse = actual amount paid (0 for pay-on-delivery,
matching today's behavior exactly)
```

## Payment — PawaPay mobile money

Grounded in PawaPay's actual v2 API (fetched from `docs.pawapay.io`, not
guessed):

- **Initiate**: `POST /v2/deposits` — sandbox `api.sandbox.pawapay.io`,
  prod `api.pawapay.io`. Bearer token auth. Body:
  `{depositId (uuid), payer:{type:"MMO", accountDetails:{phoneNumber, provider}}, amount, currency, clientReferenceId, metadata[]}`.
  Response is a synchronous ack only — `ACCEPTED` / `DUPLICATE_IGNORED` /
  `REJECTED` — never the final payment result.
- **Check status**: `GET /v2/deposits/{depositId}` →
  `ACCEPTED → PROCESSING → IN_RECONCILIATION → COMPLETED | FAILED`.
- **DRC providers**: `VODACOM_MPESA_COD`, `AIRTEL_COD`, `ORANGE_COD`,
  currency `CDF` (matches AROM's "FC" pricing) or `USD`.
- **Not yet confirmed**: the exact webhook/callback payload shape — the
  docs excerpt reachable during this research didn't include it. Must be
  verified against a real sandbox account before it's load-bearing.

### Architecture

PawaPay's Bearer token is a secret and can never reach the browser, so
initiating a payment can't be a plain client → Firestore write like the
rest of this app. The fix doesn't require Cloud Functions, though — this
app already runs server routes on the same Nitro/Cloudflare Workers
deploy (`AROM-Production/src/routes/[.mcp]/`). A new
`/api/pawapay/initiate` server route holding the token as a Workers
secret fits the existing shape.

Recording the *final* status is the one real fork, and it's the same
shape as the Sprint 3 admin-invite fork (see roadmap #10):

- **Client-side polling (chosen for this phase)** — the browser calls a
  server route that proxies "check status" (keeping the token
  server-side), and on `COMPLETED` the browser itself writes the payment
  status to its own order under a loosened rule scoped to that field. No
  new infra, no service account in the Worker. Weak point: relies on the
  tab staying open or being reopened to reconcile — acceptable for a
  stub/testing phase, revisit if it causes real support load.
- **Server-side webhook (deferred)** — reliable regardless of the
  browser, but needs the Firebase Admin SDK (a service account) inside
  the Cloudflare Worker — the same infra step being deferred for admin
  invites. Natural to solve both at once if/when that line gets crossed.

### Coexistence, not replacement

PawaPay is one of two payment paths a partner can choose at checkout:
"Payer par mobile money" or "Payer à la livraison" (today's unchanged
manual/cash flow, still reconciled by hand in the ventes journal). Nothing
about the existing order→`ventes` bridge changes for the pay-on-delivery
path.

### Proposed data model addition

```
orders/{id}
  ...existing fields...
  payment?: {
    method: "pawapay" | "cash_on_delivery"
    status: "pending" | "processing" | "completed" | "failed"
    depositId?: string        // PawaPay's UUID, when method is pawapay
    provider?: string         // VODACOM_MPESA_COD | AIRTEL_COD | ORANGE_COD
    amount?: number
    currency?: string         // "CDF"
    updatedAt?: string
  }
```

One embedded object, not a new collection — mirrors how `orders` already
embeds `items[]`. If retries need their own history later (failed attempt
→ retry with a new `depositId`), revisit as an array of attempts; not
needed for a v1.

## Sequencing

Roughly in this order, one PR per sprint (see the
[Sprint Brief](https://claude.ai/code/artifact/0a6920c5-dd67-449c-a168-8fed9eea354a)
for the prompt template):

1. ~~Account-lockout / signup session-clobber fixes~~ — done, see
   [Teddmab/AROM-Production#5](https://github.com/Teddmab/AROM-Production/pull/5).
2. Admin catalog UI: create/edit/deactivate products, image upload to
   Storage (closes roadmap #6 and the disconnect noted above).
3. Promo banner: admin-editable headline + product reference + active
   dates, shown on `/storefront`.
4. Storefront order tracking: surface delivery-relevant detail beyond the
   current pending/confirmed/fulfilled/cancelled status (ad-hoc delivery
   date set by admin on confirm, per earlier decision — no fixed
   recurring schedule yet).
5. PawaPay stub integration, per the architecture above.
6. Admin/staff invite-link signup (roadmap gap #7) — needs the Cloud
   Functions decision either standalone or bundled with the PawaPay
   webhook upgrade.
