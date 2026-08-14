# Roadmap / known gaps

Roughly in priority order.

See [flows.md](flows.md) for the role-by-role, page-by-page flow map this
roadmap is scoped against and the PawaPay payment plan, and
[sprints/v1-mvp.md](sprints/v1-mvp.md) for that flow broken into
individual, PR-sized sprints.

## Done

- **Order fulfillment → `ventes` bridge** (was #5 below). Marking a
  storefront order "Livrée" in the dashboard's Commercialisation section
  now atomically closes the order and writes one `ventes` row per line
  item — storefront sales show up in commercial KPIs immediately. See
  [data-model.md](data-model.md#order--ventes-bridge).
- **Account-lockout redirect loop.** Deactivating an admin/staff account
  didn't actually lock them out (infinite `/dashboard ↔ /login` bounce);
  a deactivated partner hit a dead-end "Redirection…" screen with no
  explanation. See
  [Teddmab/AROM-Production#5](https://github.com/Teddmab/AROM-Production/pull/5).
- **Removed all Lovable coupling from `AROM-Production`.** See
  [architecture.md](architecture.md#update-2026-08-14-lovable-removed) —
  this is what makes #11 below urgent rather than hypothetical.
- **Invite-link admin/staff signup** (part of #7 below). Turned out not
  to need the Cloud Functions decision it was originally scoped
  against — see [rbac.md](rbac.md#how-adminstaff-accounts-are-provisioned)
  for how a pure `firestore.rules` cross-document check does the whole
  job. See
  [Teddmab/AROM-Production#7](https://github.com/Teddmab/AROM-Production/pull/7).
- **Guided boutique onboarding.** `/storefront/signup` was a three-field
  form with no phone, address, or verification info — not enough to
  actually deliver to or vet a new boutique. Now a 5-step wizard
  collects contact info, a structured delivery address, and an optional
  KYC field, and denormalizes phone/address onto each order so the
  dashboard shows what's needed to fulfill it. See
  [sprints/13](sprints/13-guided-boutique-onboarding.md) and
  [data-model.md](data-model.md#guided-boutique-onboarding-sprint-13).
- **Payment on the storefront** (was #6c below). `/storefront` is now
  tabbed (Catalogue / Mes commandes) and "Commander" opens a checkout
  sheet where the partner chooses mobile money (PawaPay, stubbed — no
  real credentials yet) or cash on delivery; the order→`ventes` bridge
  now writes the real paid amount instead of a hardcoded `encaisse: 0`.
  Server-side webhook confirmation is still deferred (needs Cloud
  Functions, see #10 below). See
  [sprints/08](sprints/08-pawapay-payment-stub.md) and
  [data-model.md](data-model.md#payment-sprint-08-stub-phase).
- **Admin UI for the storefront catalog** (was #6 below — this had
  already shipped in [sprints/05](sprints/05-admin-catalog-management.md)
  but this list was never updated to reflect it). Admin/staff can add,
  price, deactivate, photograph, and (sprint 14) describe a product from
  the dashboard.
- **Promo banner** (was #6b below). A single active-or-not promo, set by
  admin, shown live at the top of `/storefront`. See
  [sprints/06](sprints/06-promo-banner.md).
- **Storefront order tracking** and **self-service profile/order/product
  detail**. A partner can now see and correct their own KYC/delivery
  info at `/storefront/profile`, see a real admin-set delivery date on
  their order, and open a detail sheet for any order or product instead
  of a bare list. See [sprints/07](sprints/07-storefront-order-tracking.md)
  and [sprints/14](sprints/14-storefront-self-service.md).

## Security & correctness

1. **Staff data-level scoping.** `firestore.rules` currently grants any
   `staff` account read/write on *all* internal ERP collections regardless
   of their `menus`. Menu scoping is UI-only today (see
   [rbac.md](rbac.md)). Fine for a small trusted team; worth tightening
   with per-collection role checks (or a `menus`-aware rule) before staff
   headcount grows past "everyone trusts everyone."
2. **No email verification / password reset flow** in the UI. Firebase
   Auth supports both; `signInWithEmailAndPassword` /
   `createUserWithEmailAndPassword` are wired but nothing calls
   `sendPasswordResetEmail` or `sendEmailVerification` yet.
3. **No rate limiting on `/storefront/signup`.** It's a public form backed
   directly by `createUserWithEmailAndPassword`; add App Check or a
   Cloud Function trigger if abuse becomes a concern.
4. **The generated admin password was communicated once, in plaintext,
   in this session's output.** It should be rotated on first login if it
   hasn't been already.

## Product

6d. **`clients` (internal registry) and `users` (partner accounts) are
   disconnected systems** — a storefront order never creates a `clients`
   doc, and `ventes.idClient` is free text, not a foreign key. Not
   urgent at current scale; worth knowing before it isn't.
7. **No admin UI for *managing* existing accounts.** *Creating* an
   admin/staff account no longer needs the CLI (invite-link signup, see
   Done above) — but listing accounts, deactivating one, or changing a
   role still does (`AROM-Backend/scripts/list-users.mjs` + a manual
   Firestore console edit). A "Personnel" admin screen for that would
   close the loop.
8. **`reset()` in the ERP store is now a no-op** (with a toast pointing to
   admin scripts) rather than actually clearing data — intentional, since
   the old "reset to seed" behavior would have wiped shared, live
   Firestore data for every user with one click. If a real "start a new
   campaign" flow is wanted, it should be a deliberate admin action
   (e.g. an Admin SDK script that archives the current campaign under a
   dated collection rather than deleting it).

## Infrastructure

9. **Budget alert.** Billing is linked to `arom-production` but no budget
   alert is configured. Set one in the GCP console
   (Billing → Budgets & alerts) — cheap insurance given usage should be
   ~$0 at this scale.
10. **Cloud Functions** would let things that genuinely need server
    compute — welcome/notification emails, the PawaPay webhook upgrade
    (see [flows.md](flows.md#architecture)) — happen without a manually-run
    script. (Account provisioning turned out *not* to need this — see
    invite-link signup in Done above.) Deliberately deferred since it
    wasn't needed for a working v1 and keeps the project off any
    Functions-related billing surface until it's actually wanted.
11. **Nothing currently deploys `AROM-Production`.** Was entirely owned by
    Lovable's pipeline; the project has been disconnected from Lovable
    (2026-08-14, see [architecture.md](architecture.md#update-2026-08-14-lovable-removed)).
    An independent Cloudflare Workers pipeline is now built (sprint 12,
    [Teddmab/AROM-Production#11](https://github.com/Teddmab/AROM-Production/pull/11))
    but not yet live — it needs `CLOUDFLARE_API_TOKEN` and
    `CLOUDFLARE_ACCOUNT_ID` as GitHub repo secrets before pushes to
    `main` actually deploy. See
    [runbook.md](runbook.md#frontend-deploys) for exactly what's needed.

## From the 2026-08-14 kickoff meeting (client: Ethical Mine)

Client walkthrough of the current build. Two things confirmed as
already satisfying what was asked for, two new asks captured as
sprints, and a longer-term vision worth recording so it isn't lost.

**Already satisfied by shipped/in-review work:**
- "Améliorer commandes" (better order management, visible delivery
  status/date) — this is exactly [sprints/07](sprints/07-storefront-order-tracking.md)
  and [sprints/14](sprints/14-storefront-self-service.md)'s order detail
  sheet, both in review as of this meeting.
- Google/Facebook login, new boutiques landing on the catalog after
  signup, separate catalog/orders views — all already shipped
  (sprint 10, and the catalogue/orders tabs work).

**New asks, captured as sprints:**
- [sprints/15](sprints/15-whatsapp-notifications.md) — WhatsApp
  notifications on order confirm/fulfill.
- [sprints/16](sprints/16-boutique-verification.md) — a simple,
  human-in-the-loop ("phone call") verification flag for new boutiques,
  distinct from and in addition to the KYC *data collection* sprint 13
  already shipped.

**"Mise en ligne" (go live)** is sprint 12 — the Cloudflare Workers
pipeline is built and waiting on the client to provide
`CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID`. Worth flagging back to
the client directly since they asked for this as a next step and it's
blocked on them, not on build work.

**Longer-term vision (explicitly "nice to have," not scheduled):**
- **Approvisionnement → Mombongo integration.** Link AROM's supply data
  to a Mombongo administrator account so the ~25 partner
  plantations' data syncs automatically instead of manual entry, with
  an "import" counterpart to the existing ERP export (import from
  Excel, from Mombongo, or from Agroconnect). The client was explicit
  this depends on Mombongo's own maturity and AROM's own process
  simplification first — "pas pour tout de suite."
- **Auto-populate production from approvisionnement.** Today both are
  filled by hand; quantities already entered at the approvisionnement
  step could pre-fill the matching production entry. Mentioned as
  something already being investigated (data-schema study), not yet a
  committed design.
- **Agroconnect integration**, positioned as helping other juice
  producers ("transformateurs") source raw pineapples, not something
  AROM itself needs directly.
- **Geographic expansion** (Kinshasa, Lubumbashi, then possibly Rwanda,
  Tanzania) — the underlying reason the Mombongo/Agroconnect
  integrations matter at all: the same approvisionnement flow should
  work for a new AROM location finding new suppliers in a new region,
  not just Kasaï. No scope or timeline attached yet.
