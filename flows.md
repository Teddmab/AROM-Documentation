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
| Partner | `/storefront/signup` → `/storefront` (Catalogue / Mes commandes tabs) | catalog, cart, checkout (mobile money or cash on delivery), own orders (live via `onSnapshot`) | `products` (read), `orders` (own — create, read; cancel while pending) | no product photos, no promo banner, no real PawaPay credentials (stubbed), order status has no delivery ETA |
| Staff | `/login` → `/dashboard` | sections listed in `menus` | full read/write on all internal collections — `firestore.rules` doesn't scope by `menus` (documented, deliberate for now, see roadmap #1) | account creation/deactivation is CLI-only |
| Admin | `/login` → `/dashboard` | everything | everything, incl. the order→`ventes` bridge | no catalog UI, no promo UI, no server-side payment webhook (client-side polling only, see below) |

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
                                                  checkout: mobile money (PawaPay,
                                                  stubbed) or cash on delivery
                                                              │
                                                              ▼
                                                     orders (pending)
                                                              │
                                              admin confirms → fulfills
                                                              │
                                                              ▼
                                    ventes row(s) created (encaisse = amount actually
                                    paid — see data-model.md#payment-sprint-08-stub-phase)
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

The payment leg of this shipped in sprint 08 (see below); the promo
banner and product-photo pieces are still pending (roadmap #6, #6b).

```
Admin publishes/edits products (name, price, photo, active flag)
        │
        └─ optional promo banner (headline + product ref + dates)   ← not yet built
                 │
                 ▼
Partner sees live catalog with photos + banner on /storefront
                 │
        chooses: pay by mobile money (PawaPay) OR pay on delivery   ← shipped
                 │
                 ▼
Order carries a real payment status, not just pending/confirmed/fulfilled
                 │
        admin confirms → fulfills
                 │
                 ▼
ventes row created with encaisse = actual amount paid (both paths now
collect the real amount — see data-model.md#payment-sprint-08-stub-phase)
```

## Payment — PawaPay mobile money (shipped, stub phase)

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

### Architecture (as shipped)

PawaPay's Bearer token is a secret and can never reach the browser, so
initiating a payment can't be a plain client → Firestore write like the
rest of this app. This doesn't require Cloud Functions, though — two
TanStack Start server functions (`createServerFn`, in
`AROM-Production/src/lib/payments/pawapay.ts`) keep the token
server-side without a dedicated Nitro `/api` route. (The original plan
here assumed the `[.mcp]` server-route pattern, since removed in the
Lovable-dependency cleanup — sprint 11 — before this sprint landed.)

Recording the *final* status is the one real fork, and it's the same
shape as the Sprint 3 admin-invite fork (see roadmap #10):

- **Client-side polling (chosen for this phase)** — the browser calls a
  server function that proxies "check status" (keeping the token
  server-side) a few times with a delay between calls. Unlike the
  originally-sketched design, the client does **not** write a payment
  status onto an already-created order under a loosened rule — instead
  the `orders` document is only created once the outcome is known
  (`COMPLETED`), so `firestore.rules` needed no changes at all. Weak
  point: relies on the tab staying open through the poll loop — if it
  closes mid-payment, no order is created even though PawaPay may have
  taken the money; acceptable for a stub/testing phase with no real
  money moving yet, revisit before real credentials go live.
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

### Data model addition (shipped)

See [data-model.md](data-model.md#payment-sprint-08-stub-phase) for the
canonical, current shape. Simpler than originally sketched: since the
order is only ever created after the outcome is known, `payment.status`
only ever needs `"pending"` (cash on delivery, until marked "livrée") or
`"completed"` (mobile money) — there's no `"processing"` / `"failed"` on
the stored document, because an order is never created for a deposit
that ends up in either of those states.

One embedded object, not a new collection — mirrors how `orders` already
embeds `items[]`. If retries need their own history later (failed attempt
→ retry with a new `depositId`), revisit as an array of attempts; not
needed for a v1.

## Sequencing

Tracked as individual sprint files, one PR per sprint — see
[sprints/v1-mvp.md](sprints/v1-mvp.md) for the full milestone (this flow,
working end to end) and [sprints/README.md](sprints/README.md) for how a
sprint file is structured and how it maps to the
[Sprint Brief generator](https://claude.ai/code/artifact/0a6920c5-dd67-449c-a168-8fed9eea354a).
