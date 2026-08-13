# Roadmap / known gaps

Roughly in priority order.

## Done

- **Order fulfillment → `ventes` bridge** (was #5 below). Marking a
  storefront order "Livrée" in the dashboard's Commercialisation section
  now atomically closes the order and writes one `ventes` row per line
  item — storefront sales show up in commercial KPIs immediately. See
  [data-model.md](data-model.md#order--ventes-bridge).

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

6. **No product photos.** `storage.rules` reserves `products/**`; the
   catalog UI and `products` docs don't have an image field wired up yet.
7. **No admin UI for account management.** Roles/menus are managed via
   CLI scripts (`AROM-Backend/scripts/`) — fine for a small team, but a
   "Personnel" admin screen (list/deactivate/change role) would remove the
   need to touch the CLI for routine changes.
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
10. **Cloud Functions** would let account provisioning (custom claims,
    welcome emails) and order notifications happen server-side instead of
    via manually-run scripts — deliberately deferred since it wasn't
    needed for a working v1 and keeps the project off any Functions-related
    billing surface until it's actually wanted.
11. **Frontend deploy path is entirely owned by Lovable.** If AROM ever
    needs a deploy pipeline independent of Lovable (e.g. a Cloudflare API
    token in `AROM-Production`'s own CI), that's a deliberate decision to
    make, not something to bolt on silently — see
    [architecture.md](architecture.md#5-cicd).
