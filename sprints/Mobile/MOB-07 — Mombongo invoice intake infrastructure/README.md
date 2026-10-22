# MOB-07 — Mombongo payment gateway proxy (revised against the real API contract)

> **Status: Mostly done** (verified 2026-09-01). Code-complete and
> provisioned: `AROM-Production/src/lib/payments/{mombongo,mombongoSigning,mombongoSystemAuth,exchangeRate,mombongoConfig,mombongoHarvest}.ts`,
> the `/api/mombongo/*` and `/api/webhooks/mombongo` routes, and
> `AROM-Backend/firestore.rules`'s `isMombongoWebhook()` predicate all
> exist; the AROM-side system account and partner config are really
> provisioned (see the update below). Two things remain open, both
> operational rather than missing code: deploying the Cloudflare Worker
> secrets (`wrangler secret put`) since `AROM-Production` isn't live yet
> (see [runbook.md](../../../runbook.md#frontend-deploys)), and
> `AROM-Production` still has no test framework at all — flagged
> honestly in this doc's own history, not hidden.

## Update (2026-08-31, later) — real implementation, provisioned, wired, and the last blocker closed

Since the previous update: partner config moved out of Worker env vars
into `externalIntegrations/mombongo` (admin-write,
`isMombongoWebhook()`-read-only — rotating a secret or onboarding a
partner never needs a deployment again); a real bug was found and fixed (`isMombongoWebhook()` had write but not read access to
`producerInvoices` — the webhook's own lookup query would have failed in
production); the AROM-side system account was actually provisioned
(`mombongo-webhook@system.arom.cd` in `arom-production-657f2`);
`createMombongoInvoice`/`createMombongoCheckout` were converted from
`createServerFn`s (internal-RPC-only) to real server routes
(`/api/mombongo/create-invoice`, `/api/mombongo/create-checkout`),
authenticated via the calling admin's Firebase ID token (verified
without the Admin SDK, via Google's `accounts:lookup` REST endpoint);
`AROM-Mobile`'s `mombongoProducerPaymentGateway.ts` now makes real
network calls instead of simulating locally, same interface as before.

**The FC→USD conversion gap (flagged, deliberately unfixed, in the
previous update) is now resolved**: `config/exchangeRate`
(`usdToCdf: 2800`), seeded from **Mombongo's own**
`config/exchange_rate.usdToCdf` — read directly from their project,
2026-08-31 — so both systems price the same transaction consistently,
not two independently-guessed numbers. `createMombongoInvoice` converts
FC/CDF → USD using this rate before calling Mombongo; a currency that's
neither still fails closed rather than guessing. **Open**: how often this
rate should be refreshed, and whether Mombongo will proactively tell AROM
when theirs changes — worth raising with them explicitly (see the
sprint's own recommendation-to-Mombongo notes).

## Revision note (2026-08-31)

This sprint's original scope assumed Mombongo would push an invoice
*into* AROM (an inbound `createExternalInvoice`-equivalent AROM would
have to build and expose publicly). **That's backwards from what
Mombongo actually built.** Their real, locally-verified contract
(`AROM Invoice Payment API`, Sprint AI on their side) has **AROM as the
caller** for both invoice registration and checkout creation —
`POST createExternalInvoice` and `POST createExternalInvoiceCheckout`
are Mombongo endpoints AROM calls, not the other way around. The only
thing that's genuinely inbound to AROM is the payment-completion
webhook (§4 of their contract). This is a smaller, safer surface than
originally scoped — one new public endpoint instead of two, and it's a
receiver, not a request-authenticator for arbitrary invoice creation.

It also turns out this doesn't need a new `mombongoInvoices` collection
at all: the natural target for a Mombongo-processed payment is AROM's
**existing** `producerInvoices` flow (MOB-06B) — an admin already
reviews and approves a producer invoice, and MOB-06B already built the
entire gateway abstraction (`ProducerPaymentGateway`,
`getPaymentGateway()`, a reserved `"real"` `PaymentGatewayMode`) for
exactly this kind of swap-in. This sprint wires that seam to Mombongo's
real API instead of inventing a parallel invoice concept.

## Implementation note (2026-08-31) — "Admin SDK" below is wrong, corrected here

Everywhere this doc says "Admin SDK," read **dedicated system Firebase
Auth account + a custom claim, not `firebase-admin`**. Research during
implementation found that the standard `firebase-admin` npm package does
not run in Cloudflare Workers at all (gRPC/Node-internals dependency,
unaffected by `nodejs_compat`) — this is *why* the server-side webhook
was deferred for so long (`flows.md#architecture` already flagged this
exact gap, just not the reason).

**What's actually built**: `AROM-Production` already ships the regular
`firebase/firestore`/`firebase/auth` *client* SDK, already initialized
SSR-safely in `src/lib/firebase/config.ts` (proven to run server-side
today, not just in theory). One dedicated, narrow-privilege Firebase
Auth account (`AROM-Backend/scripts/provision-mombongo-webhook-account.mjs`,
run once, offline — mirrors `create-user.mjs`) carries a custom claim
(`mombongoWebhook: true`). `firestore.rules` gained a narrow, additive
exception (`isMombongoWebhook()`) letting *only* that identity make the
`approuvee → paiement_en_attente → payee` transition — the existing
"no manual mark-as-paid" invariant is preserved exactly for every
ordinary `isAdmin()` account, proven by a rules-emulator test
(`AROM-Backend/tests/rules.test.mjs`, "producerInvoices —
isMombongoWebhook()"). This is a real rule-checked write, not a
privilege bypass.

## Objective

Stand up the server-side piece Mombongo's contract requires — something
that holds AROM's two Mombongo secrets (inbound-signing and
outbound-verifying, per their §2), calls their two endpoints on the
admin's behalf, and receives their completion webhook — then flips
`producerInvoices.statut` to `payee` for real, signed in as the
dedicated system account above (see implementation note) — the existing
`firestore.rules` restriction on ordinary client writes to `payee`/
`paiement_en_attente` stays exactly as-is; only this one narrow,
claim-gated identity can make that transition.

## Where this lives

Same reasoning as before: neither Expo nor a browser can hold a secret,
so this can't live inside `AROM-Mobile`. Reuse `AROM-Production`'s
existing TanStack Start server-function pattern
(`AROM-Production/src/lib/payments/pawapay.ts` is the precedent — secret
stays in `.handler()`, never ships to the browser) rather than standing
up Cloud Functions, which the project has deliberately deferred (
`AROM-Documentation/roadmap.md` #10). `AROM-Mobile` calls these routes
over plain HTTPS the same way it already calls Firebase — this is the
first time the Expo app talks to anything other than Firebase directly,
worth confirming auth (a Firebase ID token, checked server-side against
the caller's `role === "ADMIN"`) before this is built.

## Technical approach

1. **`AROM-Production/src/lib/payments/mombongo.ts`** — two server
   functions:
   - `createMombongoInvoice({ invoiceId, amountUsd, currency: "USD", reference })`
     → calls Mombongo's `POST /createExternalInvoice`, signed per their
     §2 (`x-partner-id`, `x-partner-signature`, HMAC-SHA256 over the
     exact raw body). **Currency note**: `producerInvoices.montantTotal`
     is in FC (`devise: "FC"` per the existing model); Mombongo's
     contract only accepts `amountUsd`. A conversion step is required —
     confirm the real rate source (AROM's roadmap already tracks an
     "Automated FC→USD conversion" item for the dashboard — reuse
     whatever that resolves to rather than inventing a second FC→USD
     path).
   - `createMombongoCheckout({ mombongoInvoiceId, method, phone?, operator? })`
     → calls `POST /createExternalInvoiceCheckout`. Response shape is
     already known and concrete (their §3): `card` returns a Stripe
     `clientSecret`; `mobile_money` returns `depositStatus: "ACCEPTED"`
     after requiring `phone`+`operator` in the request; `bank_transfer`
     is a `501` until they choose a provider.
2. **New server *route*, not a server function** —
   `AROM-Production/src/routes/api/webhooks/mombongo.ts`. TanStack
   Start's `createServerFn` (what `pawapay.ts` and steps 1/3 above use)
   is explicitly internal RPC from AROM's own frontend, per their own
   docs — not a stable URL a third party can POST to. A **server route**
   (`createFileRoute(...)` with a `server.handlers.POST`) is the actual
   mechanism, "meant for HTTP endpoints that need to be called from
   outside your TanStack Start application." Verifies
   `x-mombongo-signature` (their §2, a *separate* secret from the
   inbound-signing one — two secrets, not one, via `mombongoSigning.ts`'s
   shared `signHmac`/`verifyHmac`), matches `externalInvoiceId` back to a
   `producerInvoices` doc via its `mombongoInvoiceId` field, and — only
   on `status: "paid"` — transitions `producerInvoices.statut` to
   `payee`, signed in as the dedicated system account (see implementation
   note above). **Idempotent**: Mombongo's own docs say this webhook may
   arrive more than once — a repeat for an already-`payee` (or
   already-`approuvee`, meaning a prior failure already reverted it) doc
   is a 200 no-op, not a second transition or an error.
3. **`producerInvoices` model addition**: `mombongoInvoiceId?`,
   `mombongoCheckout?: { method, providerRef, clientSecret?, depositStatus?, testMode }`
   — stamped by `createMombongoCheckout` (not `createMombongoInvoice` —
   the `statut` transition and these fields are written together, in one
   claim-gated write, once a checkout genuinely exists, not at bare
   invoice-registration time), read (but never written) by the mobile
   client.
4. **Secrets**: two new Worker secrets (inbound + outbound), provisioned
   once Mombongo completes their side of §6 ("Deploy, then provision
   AROM as a partner... produces a partner id and two secrets — sent to
   you over a secure channel").
5. **`testMode`**: Mombongo's contract stamps every invoice/checkout with
   a `testMode` flag "until we agree it's time to flip it" — thread this
   through from AROM's side too (an env flag, same
   `EXPO_PUBLIC_ENABLE_MOCK_PAYMENTS`-style convention) so a simulation
   pass (this sprint's near-term goal) never risks hitting a real rail.

## Acceptance criteria

- [x] `createMombongoInvoice`/`createMombongoCheckout` correctly sign
      requests — code-complete; the actual 401/`duplicate_ignored`
      behavior is verified against their real deployment for the
      *unauthenticated* path only so far (`check-mombongo-deployment.mjs`)
      — a fully signed round-trip still needs `partners/arom` to exist.
- [x] The webhook receiver rejects a bad/missing `x-mombongo-signature`
      and is idempotent on repeated delivery — implemented and
      typechecked; not yet exercised against a real delivery (no
      provisioning yet, so Mombongo has nothing to send this URL).
- [x] A `producerInvoices` doc reaching Mombongo `status: "paid"`
      transitions to `statut: "payee"` through the dedicated system
      account only — `firestore.rules` denies the equivalent `isAdmin()`
      client write, proven by a new rules-emulator test suite, existing
      MOB-06B invariant tests still pass unchanged.
- [ ] `testMode` threading — the checkout write hardcodes `testMode: true`
      for now since Mombongo's checkout response doesn't echo the flag
      explicitly in what's documented; revisit once a real response is
      seen.
- [x] `bunx tsc --noEmit`, `bun run lint`, `bun run build`
      (AROM-Production) all pass. `firebase emulators:exec --only
      firestore,storage "npm test"` (AROM-Backend): all producerInvoices/
      isMombongoWebhook tests pass; 3 pre-existing, unrelated
      Storage-emulator cold-start failures remain (documented since
      MOB-06A).
- [ ] No test framework exists in `AROM-Production` at all (confirmed by
      search) — this pass shipped with typecheck/lint/build only, no unit
      tests for the new signing/route logic. Flagged, not silently
      skipped; bootstrapping Vitest there is real, separate scope.

## Out of scope

- The mobile wizard UI (MOB-09).
- Bank transfer (blocked on Mombongo choosing a provider — their `501`
  is authoritative, not something to work around here).
- FC↔USD conversion logic itself, if it doesn't already exist elsewhere
  — reuse, don't build a second one.

## Status update (2026-08-31)

Confirmed live: `createExternalInvoice`, `createExternalInvoiceCheckout`,
and `onExternalInvoicePaid` are deployed on `europe-west1`/`mombongo-dev`
(`firebase functions:list` against that project). Verified with
`AROM-Backend/scripts/check-mombongo-deployment.mjs` (unauthenticated
smoke check — reachability + 401 on missing/unknown-partner auth + 405
on the wrong HTTP method, all passing against the real endpoint).
**`partners/arom` does not exist yet** — AROM is not provisioned, so no
signed request can be tested. Still blocked on Mombongo running
`provisionPartner()` for AROM specifically.

**Update, same day**: the real implementation shipped —
`AROM-Production/src/lib/payments/{mombongo,mombongoSigning,mombongoSystemAuth}.ts`,
`AROM-Production/src/routes/api/webhooks/mombongo.ts`,
`AROM-Backend/scripts/provision-mombongo-webhook-account.mjs`, and the
`firestore.rules`/`rules.test.mjs` changes. Code-complete and passing
typecheck/lint/build/rules-tests; the remaining gap is exactly the same
one as before — no `partners/arom`, so nothing here has been exercised
against a real signed request yet.

**Update, later same day — AROM is provisioned.** Mombongo ran
`provisionPartner()` and handed over the two secrets directly. **The
partner id is `Arom` (capital A)**, not `arom` — confirmed against their
real `partners/` doc; every earlier reference to lowercase `arom` in
this doc, `MOMBONGO_PARTNER_ID` usage, and the Mombongo-side sprint
proposal's example snippets was a guess made before provisioning, now
known wrong. A real signed `createExternalInvoice` call succeeded
end-to-end against their live endpoint (a one-off verification script,
not committed anywhere — real `invoiceId` returned: `V1Gnd4YCIpXs4cMy9vSf`,
`testMode: true`, safe to ignore/clean up on their side). Secrets are
stored locally in `AROM-Production/.dev.vars` (gitignored, Wrangler's
own local-secret convention) — real Cloudflare deployment still needs
`npx wrangler secret put` for each, a manual step not yet done since
`AROM-Production` isn't deployed live yet (see `roadmap.md` #11).
`MOMBONGO_WEBHOOK_EMAIL`/`MOMBONGO_WEBHOOK_PASSWORD` (AROM's own side,
via `provision-mombongo-webhook-account.mjs`) are still pending —
unrelated to Mombongo's provisioning, that script hasn't been run yet.

## Dependencies / blockers

- Mombongo's §6 handoff: partner id + two secrets, not available until
  they deploy and provision AROM. Build and unit-test everything above
  against a local fixture first (mirror their own "verified against a
  local emulator, not a live URL" approach) — don't block this sprint on
  their deploy timeline.
- AROM must send Mombongo the webhook receiver's URL once it exists
  (their §6, "AROM" step) — this is the one action item on AROM's side
  their document is explicitly waiting on.
