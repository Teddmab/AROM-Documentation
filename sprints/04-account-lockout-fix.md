# Sprint 04 — Account-lockout & signup session fix

**Status:** Done — [Teddmab/AROM-Production#5](https://github.com/Teddmab/AROM-Production/pull/5)

**Rôle concerné :** Tous les rôles
**Page / zone :** `/login`, `require-role.tsx` (guards every protected page)

## Pourquoi maintenant

A deep audit of every role's path through every page (see
[flows.md](../flows.md)) found that deactivating an admin/staff account
didn't actually lock them out — an infinite `/dashboard ↔ /login` bounce
— and a deactivated partner hit a dead-end spinner with no explanation
and no way to sign out.

## Dans le périmètre

`require-role.tsx` distinguishes "wrong role for this page" (redirect to
the correct home) from "deactivated or no profile doc" (render a clear
explanation with a sign-out button — there's nowhere to redirect a
locked-out account *to*). `login.tsx` checks `profile.active` before
redirecting. `auth.tsx`'s underlying race condition (`profileResolved`
flipping one render late) that made this worse, fixed at the root.
`storefront/signup.tsx` redirects already-authenticated users instead of
letting them clobber their session.

## Hors périmètre

Password reset / email verification (roadmap #2). Rate limiting on
signup (roadmap #3).

## Décisions déjà actées

Deactivated/no-profile accounts always see an explanation screen with a
working sign-out button — never a silent redirect loop, regardless of
role.

## Contraintes

Same as sprint 01.

## Livrable

`AROM-Production` only.

## Test de fumée

- [x] Normal admin login → `/dashboard`
- [x] Normal partner signup + sign-out + re-login → `/storefront`
- [x] Login attempt on a deactivated account → clear error, signed out, no loop
- [x] Live deactivation while already on `/dashboard` → clear lockout screen, working sign-out, no loop
- [x] Authenticated admin visiting `/storefront/signup` → redirected to `/dashboard`, form never shown
