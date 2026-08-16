# Sprint 24 — Self-service "Mon profil" for admin/staff accounts

**Status:** Done — [Teddmab/AROM-Production#20](https://github.com/Teddmab/AROM-Production/pull/20)

**Rôle concerné :** Admin / Staff
**Page / zone :** Dashboard — header

## Pourquoi maintenant

"Similar to what we have from boutique, add a profile for the current
user" — the storefront partner already has a self-service profile page
(`/storefront/profile`, sprint 14) to correct their own name/contact/
address. Admin and staff accounts had no equivalent: the header showed
name/role as plain text, and renaming yourself or resetting your own
password meant an admin editing the Firestore doc by hand or the
account owner using the `/login` "Mot de passe oublié" flow, entirely
separate from the dashboard itself.

## Dans le périmètre

- **Header name/role block is now clickable.** Opens "Mon profil" —
  reusing `RecordDetailModal` (the same click-a-row detail/edit overlay
  every ERP table already uses, sprint 19) rather than a dedicated
  route, since the dashboard is a single-page section switcher, not a
  per-page router like the storefront.
- **Fields shown**: Nom affiché (editable), Email (read-only — changing
  it needs Firebase Auth's own re-auth flow, out of scope), Rôle
  (read-only), Poste (read-only, staff only, shown when set), and a
  "Mot de passe" field whose value is a button that sends a
  reset-password email via the same `resetPassword` (Firebase Auth
  hosted flow) already wired on `/login`.
- **`updateOwnProfile`** (new, `auth.tsx`) — the only self-editable
  field is `displayName`, written to both the Firebase Auth profile
  (`updateProfile`) and the `users/{uid}` Firestore doc, so the header
  and every other place `displayName` is read stay in sync immediately.

## Hors périmètre

- Email change (needs re-authentication — a bigger, separate flow) and
  in-app password change (the hosted reset-email flow already exists
  and is reused as-is, no new UI for typing a new password directly).
- Letting an account self-edit `menus`/`poste`/`role` — see Décisions
  déjà actées below.
- Mobile-viewport access to "Mon profil" — the header's account block
  (name/role and "Déconnexion") is already `sm:flex`-only (hidden below
  that breakpoint), a pre-existing gap this sprint didn't expand scope
  to close.

## Décisions déjà actées

- **Only `displayName` is self-editable, never `menus`/`poste`/`role`.**
  `firestore.rules`' `users/{uid}` self-update rule technically allows a
  signed-in account to write *any* field on its own doc as long as
  `role` itself doesn't change (see [rbac.md](rbac.md#account-lifecycle))
  — meaning a staff account could, in principle, grant itself more
  `menus` via a raw `updateDoc` call. `updateOwnProfile` is the only
  client path the UI offers, and it never touches those fields; a rules
  tightening (scoping the self-update rule to an explicit allow-list of
  fields) is a possible follow-up, not done here since no exploit path
  exists through the UI itself.
- **Reuse `RecordDetailModal`, not a new component or route.** Same
  edit-in-place shape (Modifier → inputs → Enregistrer) every other
  record in the app already uses — "Mon profil" behaves exactly like
  clicking any other row, just for the signed-in account's own doc.

## Contraintes

Frontend-only (`AROM-Production`) — no `firestore.rules` change; the
existing self-update rule already covers a `displayName`-only write.

## Livrable

`AROM-Production`: `src/lib/firebase/auth.tsx` (`updateOwnProfile`),
`src/routes/dashboard.tsx` (clickable header block, `showProfile` state,
`RecordDetailModal` wiring). `AROM-Documentation` (this file).

## Test de fumée

- [x] Clicking the header's name/role opens "Mon profil"
- [x] Email, Rôle, and (for staff) Poste show as read-only info
- [x] Editing "Nom affiché" and saving updates the header immediately
- [x] The new name persists after a page reload
- [x] "Recevoir un lien de réinitialisation" sends a password-reset email

Covered by the automated regression suite (`node regression-suite.mjs`,
58/58 passing, two consecutive clean runs).
