# Sprint 10 — Google/Facebook/email login

**Status:** Done — [Teddmab/AROM-Production#8](https://github.com/Teddmab/AROM-Production/pull/8)

**Rôle concerné :** Partenaire (signup + login) / Tous les rôles (login)
**Page / zone :** `/login`, `/storefront/signup`

## Pourquoi maintenant

Email/password was the only sign-in method. Requested directly.

## Dans le périmètre

Google and Facebook sign-in buttons on `/login` (any existing account,
any role) and `/storefront/signup` (new partners). Shared
`<OAuthButtons>` component; `auth.tsx` gets `signInWithProvider` (login —
no Firestore write, existing "no profile" handling covers a first-time
OAuth account the same way it already covers a mistyped email) and
`signUpPartnerWithProvider` (signup — creates a `role: partner` doc only
if one doesn't already exist for that `uid`).

## Hors périmètre

`/join` (invite redemption) — email/password only, by choice, not a
technical limit. A server-side webhook/callback for either provider —
not applicable, Firebase's popup flow handles the whole handshake
client-side.

## Décisions déjà actées

- Facebook App ID/Secret are configured directly in the Firebase console
  (Authentication → Sign-in method), never in app code or committed
  anywhere — see
  [runbook.md](../runbook.md#enabling-google--facebook-sign-in).
- OAuth buttons appear above the email form on both pages, with an
  "ou avec e-mail" divider — OAuth first, email as the fallback.

## Contraintes

Same as sprint 09. Full round-trip testing against real Google/Facebook
accounts isn't possible against the local emulator (no real OAuth
consent screen) — the Firebase Auth emulator's mock IdP flow (an
"Auto-generate user information" shortcut in the popup) stands in for it
and was used to verify the actual code path, not just that a popup opens.

## Livrable

`AROM-Production` only (`auth.tsx`, `components/auth/OAuthButtons.tsx`,
`routes/login.tsx`, `routes/storefront/signup.tsx`).
`AROM-Documentation` (rbac.md, runbook.md).

## Test de fumée

- [x] Both `/login` and `/storefront/signup` show Google + Facebook
      buttons above the email form
- [x] Clicking either correctly opens a real popup (verified the
      `signInWithPopup` call fires, not just that a button exists)
- [x] Full round trip via the emulator's mock IdP: Google signup creates
      a real partner account and lands on `/storefront` showing the
      correct display name
- [x] Signing in with an **existing** Google-linked account via `/login`
      routes to `/storefront` (doesn't attempt to create a second account)
- [x] Full regression suite (email login/signup, order→ventes bridge,
      catalog management, invite signup, account-lockout fix) still
      passes with zero failures after this change
