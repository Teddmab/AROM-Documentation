# Sprint 16 — Boutique verification (call-confirmation KYC)

**Status:** Done — [Teddmab/AROM-Production#12](https://github.com/Teddmab/AROM-Production/pull/12)

**Rôle concerné :** Admin (vérifie) / Partenaire (soumet, déjà fait)
**Page / zone :** Dashboard — Primes & personnel section

## Pourquoi maintenant

Raised at the 2026-08-14 kickoff (client: Ethical Mine): when a new
boutique self-registers, AROM wants a lightweight human check — a phone
call to confirm a real shop exists behind the submitted info — before
treating the account as trustworthy. Explicitly described as "KYC pas
trop dur mais KYC simple" (simple, not heavyweight).

This is a **narrower, different thing** from what sprint 13 already
shipped and explicitly scoped out. Sprint 13 built the *data collection*
(phone, address, optional CNI/RCCM) but deliberately **not** a
document-upload-and-review workflow, and **not** any admin-facing
"is this boutique verified" concept — every self-registered partner is
fully active and able to order immediately. This sprint adds the
missing piece: giving admin something to act on.

## Dans le périmètre

- A `verified` (boolean, default `false`) field on partner `users/{uid}`
  docs, set only by admin.
- An admin-facing list of boutiques — likely the first real answer to
  roadmap #7 ("no admin UI for managing existing accounts"), since a
  "boutiques to verify" list and a general accounts list are the same
  underlying screen — showing name, contact, phone, address, and a
  "Vérifié" toggle admin flips after making the confirmation call.
- Unverified boutiques are **not** blocked from ordering (see Décisions
  below) — `verified` is informational for admin, not a gate, at least
  for this pass.

## Hors périmètre

- Blocking orders/checkout until verified — not requested; would turn a
  lightweight informational flag into a hard gate with its own edge
  cases (what happens to an in-flight cart when verification is
  pending?). Revisit only if the client asks for enforcement.
- Automated verification (calling APIs, SMS OTP, document OCR) — the
  client specifically described a human phone call, not automation.
- Document upload — still explicitly out of scope, same reasoning as
  sprint 13.

## Décisions déjà actées

- **Informational flag, not a gate**, for this pass — matches "KYC
  simple" from the transcript; a hard gate wasn't asked for and would
  need product decisions (grace period? partial access?) nobody has
  made yet.
- **Piggybacks on roadmap #7** (admin account management) rather than
  being a narrow "verification-only" screen, since building a
  verification list and then a *separate* general accounts-management
  screen later would duplicate most of the same UI. Concretely, the new
  "Boutiques partenaires" card lives in the dashboard's existing Primes
  & personnel section, right next to `InviteCard` (sprint 09) — both are
  "accounts I need to act on" cards, so this keeps that grouping
  consistent rather than adding a new top-level nav section for one
  card.

## Contraintes

Same as sprint 05. `firestore.rules`: `users/{uid}` update already
allows `admin` to change any field on any user's doc (existing rule:
`allow update: if isAdmin() || (isSignedIn() && ... role unchanged)`),
so `verified` needs no rules change — confirm this before assuming
otherwise, per the standing project habit (sprint 06 got this wrong
originally and did need a rules change; sprint 13 didn't).

## Livrable

`AROM-Production` (`BoutiquesCard` in the Primes & personnel section of
`dashboard.tsx`, `verified` field on `UserProfile` in `auth.tsx`).
`AROM-Documentation` (data-model.md, roadmap.md — partially closes #7;
full admin/staff account management is still CLI-only).

## Test de fumée

- [x] New partner signup defaults to `verified: false`
- [x] Admin sees a list of boutiques with contact info and current
      verification status
- [x] Admin flips a boutique to verified — persists, visible live
- [x] An unverified partner can still browse, order, and check out
      normally (informational-only, confirmed not a gate) — implicit in
      the regression suite's ordering, since checkout happens before
      the boutique is ever verified

Covered by the automated regression suite (`node regression-suite.mjs`,
23/23 passing).
