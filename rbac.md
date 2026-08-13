# Roles & access control

Three roles, stored on `users/{uid}` in Firestore (`AROM-Backend/firestore.rules`
is the source of truth for enforcement; the frontend only mirrors it for UX).

| Role | Provisioned by | Can access | Cannot |
| --- | --- | --- | --- |
| `admin` | `AROM-Backend/scripts/create-user.mjs` only | `/dashboard`, every section, every internal Firestore collection, `/storefront` catalog | — |
| `staff` | `AROM-Backend/scripts/create-user.mjs` only | `/dashboard`, only the sections listed in their `menus` array | Sections not in `menus`; cannot create/promote other accounts |
| `partner` | Self-registers at `/storefront/signup` | `/storefront` (catalog, cart, own orders) | `/dashboard` entirely; other partners' orders; internal ERP collections |

## Why admin/staff can't self-register

`firestore.rules` only allows a client to create its own `users/{uid}` doc
when `role == "partner"`. Admin and staff accounts are provisioned
out-of-band with the Admin SDK (`create-user.mjs`), which bypasses rules
entirely — this is the boundary that keeps the storefront's public signup
form from being a path to dashboard access.

## Staff menu scoping

`menus` is either the literal string `"all"` or an array of `SectionId`
values matching `AROM-Production/src/routes/dashboard.tsx`:

```
appro | production | stock | commercialisation | marketing
| finances | personnel | kpi | parametres | roadmap | executif
```

Example — a production-floor supervisor who should only touch
approvisionnement/production/stock:

```
node scripts/create-user.mjs sup@arom.cd "..." staff "Nom" appro,production,stock
```

`canAccessMenu()` in `AROM-Production/src/lib/firebase/auth.tsx` is the
single place this is evaluated client-side (used both to filter the
sidebar and, via `<RequireRole>`, to gate the whole `/dashboard` route to
admin+staff regardless of menu).

**Important**: menu scoping today is a **UI-only** convenience, not a
data-level restriction — `firestore.rules` grants any `staff` account
read/write on all internal ERP collections (`producteurs`,
`approvisionnements`, `productions`, `stockMP`, `clients`, `ventes`,
`marketing`, `charges`). A staff account without `stock` in their `menus`
won't see the Stock UI, but could still write to `stockMP` directly via
the SDK if they inspected the app. This matches the operational reality
(all internal data is company data, not user-private data) but is worth
tightening if AROM ever has staff whose access needs to be enforced
adversarially rather than just organizationally — see
[roadmap.md](roadmap.md).

## Partner isolation

`orders/{id}` rules require `partnerId == request.auth.uid` for both read
and create, so partners only ever see their own order history — enforced
at the database layer, not just hidden in the UI. `products/{id}` is
read-only for any signed-in user (write is admin/staff only), so the
catalog is shared but never editable by a partner.

## Account lifecycle

- **Deactivating** an account: set `active: false` on their `users/{uid}`
  doc (via `list-users.mjs` + a manual Firestore console edit, or extend
  `create-user.mjs`). `firestore.rules`'s `role()` helper only returns a
  role for `active == true` users, so a deactivated account loses all
  role-gated access immediately, even with a still-valid Auth session.
- **Changing a role**: re-run `create-user.mjs` with the new role, or edit
  the Firestore doc directly (rules block a signed-in user from changing
  their *own* role, but `admin` can change anyone's).
