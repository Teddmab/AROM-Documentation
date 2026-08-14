# Sprint 09 — Invite-link admin/staff signup

**Status:** Done — [Teddmab/AROM-Backend#2](https://github.com/Teddmab/AROM-Backend/pull/2), [Teddmab/AROM-Production#7](https://github.com/Teddmab/AROM-Production/pull/7)

**Rôle concerné :** Admin (invite) / Admin & Staff (redeem)
**Page / zone :** Dashboard — Primes & personnel, new `/join` route

## Pourquoi maintenant

Admin/staff account creation was CLI-only (`AROM-Backend/scripts/create-user.mjs`).
Sprint v1-mvp.md originally scoped this out of v1 on the assumption it
needed a Cloud Functions decision (same fork as the PawaPay webhook) —
that assumption turned out to be wrong.

## Dans le périmètre

An admin creates an `invites/{id}` doc (email, role, menus) from the
dashboard; the invitee opens `/join?invite=<id>`, sees who/what they're
being invited as, and creates their own account with just a name and
password.

## Hors périmètre

Listing/deactivating/changing the role of *existing* accounts (roadmap
#7, still open) — this sprint is creation only. Automated invite
delivery (email/SMS) — the admin copies a link and shares it however
the team already communicates.

## Décisions déjà actées

- **No Cloud Functions, no Admin SDK service account.** The entire
  enforcement is a `get()` cross-reference in `firestore.rules`: a
  `users/{uid}` create with `role` in `admin`/`staff` is only allowed if
  a matching, unused, admin-issued invite exists for the email the
  caller just authenticated as, with role/menus matching exactly. This
  revises the plan in flows.md, which assumed server compute was
  required — worth remembering next time a "needs Cloud Functions"
  assumption shows up, since rules can reference other documents.
- Invite `get`-by-ID is public (the ID is the actual secret, an
  unguessable random string) so `/join` can validate before asking for a
  password; `list` is admin-only so the collection can't be enumerated.
- Only admin can create invites — staff cannot, matching rbac.md's
  existing "cannot create/promote other accounts."

## Contraintes

Same as sprint 05. Security-rules changes were verified with the
**client SDK** against the emulator, not the Admin SDK (which bypasses
rules and so can't validate a rules change).

## Livrable

`AROM-Backend` (`firestore.rules`). `AROM-Production` (dashboard invite
card, `/join` route, `auth.tsx` `getInvite`/`redeemInvite`).
`AROM-Documentation` (rbac.md, data-model.md, roadmap.md).

## Test de fumée

- [x] Admin creates an invite scoped to two dashboard sections
- [x] A fresh browser redeems the real copied link — `/join` previews
      the invited email/role before asking for anything
- [x] Redemption lands on `/dashboard` showing **exactly** the invited
      sections, not more
- [x] Admin's invite list shows it as "Utilisée" afterward
- [x] Replaying the same link shows "no longer valid," not a working form
- [x] Security-rules attack scenarios all correctly denied: non-admin
      creating an invite, self-promotion with no invite, wrong-email
      redemption, role-escalation attempt, replay of a used invite
