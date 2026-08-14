# Sprint 13 — Guided boutique onboarding

**Status:** Done — [Teddmab/AROM-Production#11](https://github.com/Teddmab/AROM-Production/pull/11)

**Rôle concerné :** Partenaire (s'inscrit) / Admin (livre, vérifie)
**Page / zone :** `/storefront/signup`, `/storefront`, Dashboard — Commercialisation

## Pourquoi maintenant

`/storefront/signup` was a single three-field form (name, email,
password) — not enough to actually deliver to or verify a boutique.
Nothing in the schema captured a phone number, a physical address, or
any identifying info; `roadmap.md` didn't even flag it as a known gap,
it simply hadn't come up yet. Requested directly: onboarding needed to
be guided (one question at a time, not a wall of fields) for
non-technical shop owners, while still collecting what AROM actually
needs to deliver and do light KYC.

## Dans le périmètre

A 5-step wizard replacing the single form — see
[data-model.md](../data-model.md#guided-boutique-onboarding-sprint-13)
for the exact step-by-step breakdown and the two-write account-creation
shape. In scope:

- Structured delivery address (ville/commune/quartier/repère), not a
  single free-text field — deliberate, since DRC addresses commonly lack
  formal street addressing.
- A contact name + phone distinct from the account email, since the
  person placing orders isn't always reachable by email.
- An optional CNI/RCCM number field — light-touch KYC.
- Denormalizing `partnerPhone`/`partnerAddress` onto each order at
  checkout time (same pattern as `items[]` snapshotting product
  name/price already), surfaced as a new "Livraison" column in the
  dashboard's `OrdersCard` — the actual reason admin now has what it
  needs to deliver correctly, not just a data-collection exercise.
- A single stubbed "point of sale" shown to the partner at the end of
  the wizard, confirming where their orders ship from.

## Hors périmètre

- **Document upload + admin review queue.** Considered and explicitly
  rejected for this sprint — would need new `storage.rules` (no
  precedent for user-uploaded files existed before this sprint) and a
  new admin review UI. KYC here is typed fields only; the account is
  usable immediately, same as before.
- **Real points-of-sale / delivery-zone matching.** AROM currently
  delivers from one location. A `pointsDeVente` collection with a real
  commune → depot lookup is deferred until there's a second depot to
  route between — see data-model.md.
- **Editing profile info after onboarding.** No "Mon compte" screen was
  added; a partner who needs to correct a typo has no self-service path
  yet (roadmap #7, admin account management, covers this eventually).
- **Real DRC administrative-division data.** The ville dropdown lists
  well-known Kasaï-region cities plus "Autre" (free text) rather than an
  authoritative commune/quartier dataset, which wasn't available to
  build against.

## Décisions déjà actées

- **Profile info only, no document upload** — explicit tradeoff between
  KYC depth and shipping speed; revisit if AROM needs stronger identity
  verification later.
- **Single depot, stubbed as a constant** — not worth a collection +
  matching algorithm for exactly one entry (`AROM_DEPOT_NAME` in
  `AROM-Production/src/lib/storefront/depot.ts`).
- **Structured ville/commune/quartier over free-text or map-pin address**
  — easiest for non-technical users on low-end phones, no maps API
  dependency, and DRC addressing conventions favor named
  administrative divisions + landmarks over precise coordinates anyway.
- **Two Firestore writes, not five** — account doc created immediately
  after auth (step 1) so a partner is never left signed-in with zero
  profile doc (which `RequireRole` treats as a dead-end lockout); the
  remaining fields are held client-side and written once at the end.

## Contraintes

No `firestore.rules` or `storage.rules` changes were needed — see
data-model.md for why. Firestore rejects literal `undefined` field
values (this project doesn't set `ignoreUndefinedProperties`), so
optional fields (`repere`, `idNumber`, `partnerPhone`, `partnerAddress`)
are conditionally spread into the write rather than set to `undefined`.

## Livrable

`AROM-Production` (`storefront/signup.tsx` rewritten as a wizard,
`auth.tsx` — extended `UserProfile`, `completePartnerOnboarding`,
`storefront/depot.ts`, `storefront/index.tsx` onboarding guard,
`CheckoutSheet.tsx` order snapshot, `dashboard.tsx` Livraison column).
`AROM-Documentation` (this file, data-model.md, rbac.md).

## Test de fumée

- [x] A fresh visitor completes all 5 wizard steps and lands on
      `/storefront`
- [x] Step 5 shows the assigned depot before confirming
- [x] Closing the tab mid-wizard and returning to `/storefront/signup`
      resumes at step 2 with the account already created, not step 1
- [x] Navigating directly to `/storefront` with onboarding incomplete
      redirects back to the wizard instead of showing the catalog
- [x] A placed order carries the partner's phone/address; the admin's
      Commercialisation → Commandes table shows it in a Livraison column
- [x] Google/Facebook signup still works and also lands in the wizard
      at step 2 (not skipped)

Covered by the automated regression suite (`node regression-suite.mjs`),
extended in this sprint to walk the full wizard — 13/13 passing.
