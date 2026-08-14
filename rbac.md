# Roles & access control

Three roles, stored on `users/{uid}` in Firestore (`AROM-Backend/firestore.rules`
is the source of truth for enforcement; the frontend only mirrors it for UX).

| Role | Provisioned by | Can access | Cannot |
| --- | --- | --- | --- |
| `admin` | `AROM-Backend/scripts/create-user.mjs`, or redeeming an admin-issued invite at `/join` | `/dashboard`, every section, every internal Firestore collection, `/storefront` catalog | — |
| `staff` | Same as `admin` | `/dashboard`, only the sections listed in their `menus` array | Sections not in `menus`; cannot create/promote other accounts (cannot create invites either) |
| `partner` | Self-registers at `/storefront/signup` | `/storefront` (catalog, cart, own orders) | `/dashboard` entirely; other partners' orders; internal ERP collections |

## How admin/staff accounts are provisioned

Two paths, both ending in the same `users/{uid}` shape:

1. **CLI** (`create-user.mjs`) — uses the Admin SDK, which bypasses rules
   entirely. Still the only way to set a specific initial password
   without the account holder choosing their own.
2. **Invite link** (`/join?invite=<id>`) — an admin creates an
   `invites/{id}` doc from the dashboard's "Primes & personnel" section
   (email, role, menus), shares the resulting link however the team
   already communicates (WhatsApp, email — no email-sending
   infrastructure involved), and the invitee redeems it themselves: they
   create their own Firebase Auth account and write their own
   `users/{uid}` doc.

The second path sounds like it should reopen the boundary that keeps the
public storefront signup from being a path to dashboard access — it
doesn't, because `firestore.rules` never trusts the *caller's choice* of
role. A `users/{uid}` create with `role` in `admin`/`staff` is only
allowed if a `get()` on the invite named in the write shows: it exists,
it's unused, its `email` matches `request.auth.token.email` (the email
the caller just authenticated as — not a caller-supplied field), and its
`role`/`menus` match exactly what's being written. No Cloud Function or
Admin SDK service account is involved; the entire enforcement is that
cross-document check, evaluated the same way for every write attempt.
Invites are single-use (`used` flips to `true` atomically with the
`users/{uid}` write) and only an admin can create one — `isAdmin()` in
`firestore.rules`, not client-side gating (the dashboard also hides the
"Inviter" card from staff, but that's UX, not the boundary).

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
- **Revoking an unused invite**: delete the `invites/{id}` doc (admin-only
  by rule) — the dashboard's invite list has a button for this. Once used,
  an invite can't be revoked or reused; deactivate the resulting account
  instead, same as any other.
- **`onboardingComplete` (partner-only, sprint 13)** is a separate axis
  from `active` — it tracks whether the guided onboarding wizard
  (contact/address/KYC fields) has been finished, not whether the
  account is enabled. A partner with `onboardingComplete: false` is
  still `active` and can sign back in, but is redirected to
  `/storefront/signup` to finish instead of reaching the catalog. See
  [data-model.md](data-model.md#guided-boutique-onboarding-sprint-13).
