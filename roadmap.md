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

6. **No admin UI for the storefront catalog at all** — not just photos.
   `products` is only ever written by `AROM-Backend/scripts/seed.mjs` or
   by hand in the Firestore console; there's no way to add, price,
   deactivate, or photograph a product from the dashboard, even though
   `storage.rules` already reserves `products/**` for images. See
   [flows.md](flows.md#how-data-circulates-today).
6b. **No promo banner.** Admin has no way to surface a promotion on
   `/storefront` — doesn't exist in any form yet.
6c. **No payment on the storefront.** Orders go straight from
   pending → confirmed → fulfilled with no payment concept; the
   order→`ventes` bridge always writes `encaisse: 0`. PawaPay mobile
   money plan (coexisting with the manual/cash flow) is in
   [flows.md](flows.md#payment--pawapay-mobile-money).
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
