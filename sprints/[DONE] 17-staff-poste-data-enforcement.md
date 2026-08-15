# Sprint 17 — Staff poste, per-person bonus tracking, real data-level enforcement

**Status:** Done — [Teddmab/AROM-Production#13](https://github.com/Teddmab/AROM-Production/pull/13), [Teddmab/AROM-Backend#4](https://github.com/Teddmab/AROM-Backend/pull/4)

**Rôle concerné :** Admin (assigne) / Staff (travaille dans son périmètre)
**Page / zone :** Dashboard — Primes & personnel, Approvisionnement/Production/Stock, Commercialisation/Marketing

## Pourquoi maintenant

Requested directly after a page-by-page audit of the dashboard surfaced
that "Directeur de Production" and "Chargée de Commercialisation"
already existed as bonus-calculation *labels* in Primes & personnel,
but weren't real accounts — one global prime/commission number was
shown regardless of who actually did the work, and `staff` menu
scoping was UI-only (flagged as roadmap #1 since the initial audit).
Explicit requirement: both the cosmetic job-title label **and**
per-person bonus tracking **and** real data-level enforcement — robust
enough to put in front of real staff immediately, but simple enough
for non-technical people to use without training.

## Dans le périmètre

- `poste` field on staff `users/{uid}` docs: `"Directeur de Production"`,
  `"Chargée de Commercialisation"`, or `"Personnalisé"`.
- Invite flow: picking a named poste auto-fills the matching `menus` —
  no manual section-picking for the common case.
- New "Équipe (staff)" admin card to assign/change poste on any
  *existing* staff account (invite-based or CLI-created).
- `productions`/`ventes` entries auto-fill `responsable`/`commerciale`
  and stamp `staffUid` from the logged-in account — removes a field
  non-technical users had to type, and gives per-person bonus tracking
  a reliable key instead of matching free-text names.
- Primes & personnel's two bonus cards are per-person tables now: one
  row per staff member holding that poste, computed only from *their*
  entries, plus a "Non attribué" line reconciling anything logged
  before this sprint or by an unscoped/admin account.
- `firestore.rules` actually enforces the split: a "Directeur de
  Production" account can only read/write `producteurs`/
  `approvisionnements`/`productions`/`stockMP`; "Chargée de
  Commercialisation" only `clients`/`ventes`/`marketing`/`products`/
  `orders`. Not just hidden in the UI.

## Hors périmètre

- Blocking/gating anything beyond data access — no new approval
  workflows.
- A manual menu editor for existing staff (StaffCard changes `poste`
  and, for named postes, resets `menus` to match; there's still no way
  to hand-pick individual sections for an existing account outside the
  invite flow — matches the pre-existing gap, not a new one).
- More than two named postes. `"Personnalisé"` remains the escape
  hatch for anything else, unchanged from before this sprint.
- Full admin/staff account management (deactivate, change role) —
  still CLI-only; this sprint only adds poste assignment. Partially
  closes roadmap #7, doesn't finish it.

## Décisions déjà actées

- **Opt-in, not a migration.** A staff account with no `poste` set (or
  `"Personnalisé"`) keeps the exact access it had before this sprint —
  full internal-collection read/write. Only assigning a specific named
  poste turns on enforcement for that one account. This was the
  explicit design goal for shipping to real production immediately:
  nothing about any existing account changes until admin deliberately
  opts it in, so there was no migration step, no risk window, no
  account that could get silently locked out.
- **`charges` (fixed costs) is admin + unscoped-staff only** — neither
  named poste obviously owns it, and there's still no UI to write to
  it at all (pre-existing gap, unrelated to this sprint).
- **`products`/`orders` fall under "Chargée de Commercialisation"**,
  not a separate storefront-specific poste — they're already presented
  together with `ventes`/`clients`/`marketing` under one
  Commercialisation menu, so enforcement follows the same boundary the
  UI already draws.
- **`staffUid`, not name-matching.** Auto-filling `responsable`/
  `commerciale` from `profile.displayName` (readable) while also
  stamping a `staffUid` (reliable) means the per-person bonus
  breakdown doesn't depend on nobody ever mistyping a name.
- **A tampering check, not just a happy-path test.** Invite redemption
  now also carries `poste` through, validated the same way `role`/
  `menus` already were — a redeemer can't self-assign a different
  poste than what was actually invited. Verified explicitly (see Test
  de fumée).

## Contraintes

Same as sprint 09 — `firestore.rules` changes verified with the
**client SDK** against the emulator, not the Admin SDK (which bypasses
rules and can't validate a rules change).

## Livrable

`AROM-Backend` (`firestore.rules` — poste-scoping functions and
per-collection rules). `AROM-Production` (`poste` on `UserProfile`,
`StaffCard`, `ProductionBonusCard`/`CommercialBonusCard`, `staffUid` on
`Production`/`Vente`, invite flow). `AROM-Documentation` (this file,
data-model.md, rbac.md, roadmap.md — partially closes #7).

## Test de fumée

- [x] Admin invites a staff member with poste "Directeur de
      Production" — their `menus` auto-fill to Approvisionnement/
      Production/Stock/Personnel, no manual picking
- [x] That staff account's dashboard sidebar shows only those sections
- [x] They log a production lot — "Responsable" is no longer a field
      to type; it shows up auto-attributed to them in the qualité table
- [x] Their per-person prime appears on Primes & personnel, computed
      from only their own lots
- [x] A staff account with no poste (or "Personnalisé") is completely
      unaffected — still full access, matching pre-sprint behavior
- [x] Admin can assign a poste to a pre-existing staff account from the
      new "Équipe" card
- [x] `firestore.rules`: a "Directeur de Production" account is denied
      writing to `ventes`/`clients`/`marketing`/`charges`; a "Chargée
      de Commercialisation" account is denied writing to `productions`/
      `approvisionnements`/`stockMP`/`charges` — 24 attack-scenario
      tests, all passing
- [x] Redeeming an invite with a tampered poste (different from what
      was actually invited) is denied

Covered by the automated regression suite (`node regression-suite.mjs`,
26/26 passing) plus a dedicated `firestore.rules` attack-scenario suite
(24/24 passing, client SDK against the emulator).
