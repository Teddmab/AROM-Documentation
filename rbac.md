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
`/storefront/signup` also offer Google and Facebook. `/join` (invite
redemption by pasting an invite link directly, no account yet) stays
email/password-only — that one's still a scope decision, not a technical
limit. `/storefront/signup` **can** now redeem an admin/staff invite too
(see below), so OAuth-based invite redemption does happen, just not
through `/join` itself.

The role-safety property is the same one invites rely on: the client
never gets to choose a role, only Firestore rules decide what a write is
allowed to contain. Notably, `isValidInviteRedemption()` only inspects
`request.auth.token.email` — it doesn't care whether that auth session
came from a password sign-up or an OAuth popup — so redeeming an invite
with an already-authenticated OAuth account needed no `firestore.rules`
change.

- **`/login`** (`signInWithProvider`) only ever calls `signInWithPopup` —
  no Firestore write at all. If the signed-in Google/Facebook account has
  no `users/{uid}` doc, the existing "no profile" handling kicks in
  (sign out, explain, point at signup) — logging in was never supposed to
  create an account, OAuth or not.
- **`/storefront/signup`** is now a 3-step flow — account (OAuth or
  email/password) → role → details:
  - Account creation only ever produces a bare Firebase Auth credential
    (`createAccount` for email/password; OAuth reuses `signInWithProvider`
    as-is) — no `users/{uid}` doc yet, since the role isn't known.
  - The role step's "Boutique partenaire" branch (`createPartnerProfile`)
    writes the `role: "partner"` doc, then continues to the delivery-
    details step (`completePartnerOnboarding`, unchanged).
  - The role step's "J'ai une invitation d'équipe" branch has the user
    paste the invite link/ID, previews it (`getInvite`, same call `/join`
    makes), and on confirm calls `redeemInviteWithCurrentUser` — which
    checks the invite's email against the already-authenticated user's
    email and, if it matches, redeems it exactly like `/join`'s
    `redeemInvite` does (same batched write, factored into a shared
    helper). A mismatch is surfaced client-side and would be rejected by
    `firestore.rules` regardless.
  - An existing user who ends up on the signup page by mistake gets
    routed to their account by the page's own resume/redirect effect, not
    a second one — same property the old single-step OAuth signup relied
    on.

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

## Poste-based data scoping (sprint 17, extended MOB-04)

A staff account can additionally carry a `poste`:
`"Directeur de Production"`, `"Chargée de Commercialisation"`,
`"Agent de collecte"`, or `"Personnalisé"`. Unlike `menus`, `poste` is
read by `firestore.rules` itself and actually restricts which
collections that account can read/write — real enforcement, not UI
hiding:

```
// AROM-Backend/firestore.rules
function poste() {
  return isStaff() ? userDoc().data.get('poste', null) : null;
}
function isProductionStaff() { return poste() == 'Directeur de Production'; }
function isCommercialStaff() { return poste() == 'Chargée de Commercialisation'; }
function isCollectionAgentStaff() { return poste() == 'Agent de collecte'; }
function isUnscopedStaff() {
  return isStaff() && !(poste() in [
    'Directeur de Production', 'Chargée de Commercialisation', 'Agent de collecte'
  ]);
}
```

| Poste | Full access to | Denied on |
| --- | --- | --- |
| Directeur de Production | `producteurs`, `approvisionnements`, `productions`, `stockMP` | `clients`, `ventes`, `marketing`, `products`, `orders`, `charges` |
| Chargée de Commercialisation | `clients`, `ventes`, `marketing`, `products`, `orders` | `producteurs`, `approvisionnements`, `productions`, `stockMP`, `charges` |
| Agent de collecte (MOB-04) | `producteurs`, `approvisionnements` | `productions`, `stockMP`, `clients`, `ventes`, `marketing`, `products`, `orders`, `charges` |
| No poste, or `Personnalisé` | Everything (unchanged from before sprint 17) | — |

**This is an opt-in per account, never a migration.** Every staff
account that existed before sprint 17 — and every new one that doesn't
get a poste assigned — keeps full access. Only assigning a specific
named poste (at invite time, or afterward from the dashboard's "Équipe
(staff)" card) turns on scoping for that one account. `charges` (fixed
costs) is excluded from every named poste — none obviously owns it,
and there's still no UI to write to it at all.

`Agent de collecte` exists specifically for the mobile app's
Agent-de-collecte role (`AROM-Mobile/src/features/auth/roleAuthorization.ts`
— see that file's own doc comment): before MOB-04, a mobile collection
agent needed the broader `Directeur de Production` poste (or an
unscoped account) to write receptions, which also granted
`productions`/`stockMP` access it never needed. `producteurs`/
`approvisionnements` also gained real field/type validation on
`create`/`update` as part of MOB-04 (previously a blanket
`allow read, write` with no shape check at all, unlike `users`/
`invites`) — see `AROM-Backend/firestore.rules`'s
`isValidProducteur`/`isValidApprovisionnement`. This closes a data-
integrity gap but is not, and cannot be, a guarantee that a client
never reuses a stable document id across two logically different
records — that's a client-side id-generation property (collision-
resistant UUIDs), not something Firestore rules can verify.

Verified with 24 attack-scenario tests against the emulator using the
**client SDK** (Admin SDK bypasses rules and can't validate a rules
change): each poste denied on every collection outside its own,
unscoped/`Personnalisé`/admin accounts fully unaffected, and a redeemer
can't tamper their way into a different poste than what an invite
actually granted (the same cross-document check pattern as
`isValidInviteRedemption()` above, extended to cover `poste`).

MOB-04's `Agent de collecte` poste, the new field/type validation on
`producteurs`/`approvisionnements`, and the `approvisionnements`
Storage evidence-photo path are covered by a separate, committed suite:
`AROM-Backend/tests/rules.test.mjs` (`npm run test:rules` — a plain
`firebase emulators:exec --only firestore,storage "npm test"` starts the
emulators under `.firebaserc`'s *default* project, not the
`arom-rules-test` project this suite's data lives under, which silently
breaks every Storage-rules cross-service `firestore.get()` call; see
`test:rules`'s own `--project arom-rules-test` flag and the doc comment
at the top of `rules.test.mjs`). It does not re-derive the original
sprint-17 suite referenced above, which has no committed record in this
repo to run from.

## Task (Consignes) stage ownership

Firestore's `tasks/{id}` collection is a distinct, newer surface from the
poste-scoped collections above — it deliberately has **no** pre-sprint-17
grandfather clause. A missing poste, `"Personnalisé"`, or any unrecognized
poste value is denied every stage, full stop, even though that same
account keeps its own unscoped access to every *other* collection above.
This is the one authoritative mapping both `AROM-Backend/firestore.rules`'
`canReadTaskStage()`/`canCreateTaskStage()` (the real security boundary)
and each app's own presentational mirror — `AROM-Production`'s
`visibleTaskStages()` (`src/lib/erp/tasks.ts`) and `AROM-Mobile`'s
`VISIBLE_TASK_STAGES_BY_ROLE` (`src/features/sync/syncPlan.ts`) — are
hand-kept in sync with, since neither TypeScript codebase can literally
import a `.rules` file. **When a stage's ownership changes, update all
three together; a mismatch between this table and the code is a bug.**

| Stage | Created by (actor) | Originating business event | Responsible poste | Read/update | Resolution condition |
| --- | --- | --- | --- | --- | --- |
| `production` | Admin, unscoped staff, Directeur de Production, or Agent de collecte (whoever can write `approvisionnements`) | A new réception (raw-fruit purchase) is recorded | Directeur de Production (primary), Agent de collecte (visibility only — see below) | Admin, unscoped, Directeur de Production, Agent de collecte | A real `productions/{id}` lot is created from the réception (or a legacy status flip if the source réception can no longer be resolved) |
| `stock` | Admin, unscoped staff, or Directeur de Production — chained automatically when completing a `production` task | A production lot's completion also consumes raw-material stock | Directeur de Production | Admin, unscoped, Directeur de Production | A real `stockMP` "Sortie" (stock-out) document is written |
| `commercialisation` | Same actors as `stock`, created in the same action | The same production lot's output becomes sellable inventory | Chargée de Commercialisation (the task is a handoff — production creates it, commercial staff resolve it) | Admin, unscoped, Chargée de Commercialisation | A real `ventes` (sale) document is written |
| `kyc` | The partner themselves, for their own account, at onboarding completion | A storefront partner finishes their onboarding form | Chargée de Commercialisation | Admin, unscoped, Chargée de Commercialisation | `users/{uid}.verified` is set to `true` |
| `order-confirm` | The partner themselves, for their own order, at checkout | A storefront partner places an order | Chargée de Commercialisation | Admin, unscoped, Chargée de Commercialisation | The order's `status` is set to `"confirmed"` |
| `order-fulfill` | Admin, unscoped staff, or Chargée de Commercialisation — chained automatically when completing an `order-confirm` task | A confirmed order is ready to prepare/deliver | Chargée de Commercialisation | Admin, unscoped, Chargée de Commercialisation | The order's `status` is set to `"fulfilled"` and a real `ventes` document is written |
| `invite` | Admin only | An admin sends a team invite | Admin only — no operational poste ever legitimately creates or resolves this | Admin only | A manual "mark done" click, once the admin has confirmed the invitee joined (not automatically verified against the invite's own `used` flag — a known, accepted limitation, not a security gap: nothing sensitive hinges on this stage being marked done promptly) |

**Why Agent de collecte sees `production` tasks.** A new réception is
this poste's own real, everyday action — the task is a reminder that the
réception it just logged still needs to become a production lot. This
poste cannot itself complete that task (`productions/{id}` write access is
Directeur de Production/admin/unscoped only), so its access here is
read-only visibility into the funnel it feeds, not a completion right.

**Why `kyc` is Chargée de Commercialisation-only.** An earlier version of
this mapping (and of `canReadTaskStage`) gave `kyc` to Directeur de
Production and Agent de collecte too. A 2026-09 stage-ownership audit
found that was an accidental default — every staff account could read
`users` regardless of poste, and every named poste's own web menus happen
to include `"personnel"` (where the partner directory already lives) — not
a real job-function need. Verifying a partner's onboarding identity has no
connection to production or collection logistics; it *is* directly
relevant to Chargée de Commercialisation, who processes that partner's
orders once they're verified. Corrected in `firestore.rules`,
`visibleTaskStages()`, and `VISIBLE_TASK_STAGES_BY_ROLE` together.

**Why Directeur de Production is authorized for `COLLECTION_AGENT` mode on
mobile.** This is intentional, not a mistake: this poste's real web menus
(`taches, appro, production, stock, personnel`) already include `appro`
(raw-fruit procurement) — the same collections `AROM-Mobile`'s
COLLECTION_AGENT mode covers. A Directeur de Production account is a
strict superset of what an Agent de collecte account can do, so
`AROM-Mobile/src/features/auth/roleAuthorization.ts` authorizing both
COLLECTION_AGENT and PRODUCTION for this poste is a correct derivation
from real, unchanged web access, not a broader grant invented for mobile.

**Task-creation is capped at each stage's underlying business record.**
`canCreateTaskStage()` never grants create access wider than the accounts
that can already write the real record that stage's completion produces
or chains from (e.g. `stock`/`commercialisation` creation matches
`productions/{id}`'s own write rule) — `invite` is the one exception, with
no underlying business record at all, so it stays admin-only with no
poste-based allowance whatsoever. This closed a real gap found during the
2026-09 audit: the previous `create` rule was `isAdminOrStaff()` with no
stage check, so any staff account — including an unscoped/`Personnalisé`/
no-poste one — could write a `stage: "invite"` task directly via the SDK,
bypassing both apps' admin-only UI gate (`InviteCard` on web,
`team-invite.tsx` on mobile), neither of which was ever backed by a
matching rule.

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
