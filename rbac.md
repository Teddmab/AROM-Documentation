# Roles & access control

Three roles, stored on `users/{uid}` in Firestore (`AROM-Backend/firestore.rules`
is the source of truth for enforcement; the frontend only mirrors it for UX).

| Role | Provisioned by | Can access | Cannot |
| --- | --- | --- | --- |
| `admin` | `AROM-Backend/scripts/create-user.mjs`, or redeeming an admin-issued invite at `/join` | `/dashboard`, every section, every internal Firestore collection, `/storefront` catalog | — |
| `staff` | Same as `admin` | `/dashboard`, only the sections listed in their `menus` array | Sections not in `menus`; cannot create/promote other accounts (cannot create invites either) |
| `partner` | Self-registers at `/storefront/signup` — email/password, Google, or Facebook | `/storefront` (catalog, cart, own orders) | `/dashboard` entirely; other partners' orders; internal ERP collections |

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

## Sign-in methods

Every role can sign in with email/password. `/login` and
`/storefront/signup` also offer Google and Facebook — `/join` (invite
redemption) does not, by scope decision, not a technical limit.

The role-safety property is the same one invites rely on: the client
never gets to choose a role, only Firestore rules decide what a write is
allowed to contain.

- **`/login`** (`signInWithProvider`) only ever calls `signInWithPopup` —
  no Firestore write at all. If the signed-in Google/Facebook account has
  no `users/{uid}` doc, the existing "no profile" handling kicks in
  (sign out, explain, point at signup) — logging in was never supposed to
  create an account, OAuth or not.
- **`/storefront/signup`** (`signUpPartnerWithProvider`) calls
  `signInWithPopup`, then creates a `role: "partner"` doc **only if** one
  doesn't already exist for that `uid` — the exact same
  `request.resource.data.role == 'partner'` rule email/password signup
  already relies on. An existing user who ends up on the signup page by
  mistake gets routed to their account, not a second one.

Provider setup (Google, Facebook) happens entirely in the Firebase
console, never in app code — see
[runbook.md](runbook.md#enabling-google--facebook-sign-in).

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

**Important**: menu scoping via `create-user.mjs`/manually-picked
`menus` (i.e. no `poste`, or `poste: "Personnalisé"`) is still
**UI-only**, not a data-level restriction — a staff account without
`stock` in their `menus` won't see the Stock UI, but could still write
to `stockMP` directly via the SDK if they inspected the app. This
matches the operational reality (all internal data is company data,
not user-private data) for anyone not opted into a specific poste. For
staff assigned `"Directeur de Production"` or `"Chargée de
Commercialisation"`, this is no longer true — see below.

## Poste-based data scoping (sprint 17)

A staff account can additionally carry a `poste`:
`"Directeur de Production"`, `"Chargée de Commercialisation"`, or
`"Personnalisé"`. Unlike `menus`, `poste` is read by `firestore.rules`
itself and actually restricts which collections that account can
read/write — real enforcement, not UI hiding:

```
// AROM-Backend/firestore.rules
function poste() {
  return isStaff() ? userDoc().data.get('poste', null) : null;
}
function isProductionStaff() { return poste() == 'Directeur de Production'; }
function isCommercialStaff() { return poste() == 'Chargée de Commercialisation'; }
function isUnscopedStaff() {
  return isStaff() && !(poste() in ['Directeur de Production', 'Chargée de Commercialisation']);
}
```

| Poste | Full access to | Denied on |
| --- | --- | --- |
| Directeur de Production | `producteurs`, `approvisionnements`, `productions`, `stockMP` | `clients`, `ventes`, `marketing`, `products`, `orders`, `charges` |
| Chargée de Commercialisation | `clients`, `ventes`, `marketing`, `products`, `orders` | `producteurs`, `approvisionnements`, `productions`, `stockMP`, `charges` |
| No poste, or `Personnalisé` | Everything (unchanged from before sprint 17) | — |

**This is an opt-in per account, never a migration.** Every staff
account that existed before sprint 17 — and every new one that doesn't
get a poste assigned — keeps full access. Only assigning a specific
named poste (at invite time, or afterward from the dashboard's "Équipe
(staff)" card) turns on scoping for that one account. `charges` (fixed
costs) is excluded from both named postes — neither obviously owns it,
and there's still no UI to write to it at all.

Verified with 24 attack-scenario tests against the emulator using the
**client SDK** (Admin SDK bypasses rules and can't validate a rules
change): each poste denied on every collection outside its own,
unscoped/`Personnalisé`/admin accounts fully unaffected, and a redeemer
can't tamper their way into a different poste than what an invite
actually granted (the same cross-document check pattern as
`isValidInviteRedemption()` above, extended to cover `poste`).

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
- **Self-updating your own profile (sprint 24)**: the dashboard's "Mon
  profil" (click the header's name/role) lets any signed-in admin/staff
  account rename itself and trigger a password-reset email. The
  self-update rule only blocks a `role` change — it would technically
  allow a broader write (`menus`, `poste`, etc.) too, but
  `updateOwnProfile` (`AROM-Production/src/lib/firebase/auth.tsx`) is
  the only client path offered, and it only ever writes `displayName`.
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
- **`verified` (partner-only, sprint 16)** is a third, independent axis —
  whether admin has confirmed the boutique is real with a phone call.
  Purely informational: it doesn't gate `active`-equivalent access the
  way `active`/`onboardingComplete` do, and no `firestore.rules` check
  reads it. Admin flips it from the dashboard's "Boutiques partenaires"
  card (Primes & personnel section).
