# Roadmap / known gaps

Roughly in priority order.

See [flows.md](flows.md) for the role-by-role, page-by-page flow map this
roadmap is scoped against and the PawaPay payment plan, and
[sprints/v1-mvp.md](sprints/v1-mvp.md) for that flow broken into
individual, PR-sized sprints.

## Done

- **Order fulfillment → `ventes` bridge** (was #5 below). Marking a
  storefront order "Livrée" in the dashboard's Commercialisation section
  now atomically closes the order and writes one `ventes` row per line
  item — storefront sales show up in commercial KPIs immediately. See
  [data-model.md](data-model.md#order--ventes-bridge).
- **Account-lockout redirect loop.** Deactivating an admin/staff account
  didn't actually lock them out (infinite `/dashboard ↔ /login` bounce);
  a deactivated partner hit a dead-end "Redirection…" screen with no
  explanation. See
  [Teddmab/AROM-Production#5](https://github.com/Teddmab/AROM-Production/pull/5).
- **Removed all Lovable coupling from `AROM-Production`.** See
  [architecture.md](architecture.md#update-2026-08-14-lovable-removed) —
  this is what makes #11 below urgent rather than hypothetical.
- **Invite-link admin/staff signup** (part of #7 below). Turned out not
  to need the Cloud Functions decision it was originally scoped
  against — see [rbac.md](rbac.md#how-adminstaff-accounts-are-provisioned)
  for how a pure `firestore.rules` cross-document check does the whole
  job. See
  [Teddmab/AROM-Production#7](https://github.com/Teddmab/AROM-Production/pull/7).
- **Guided boutique onboarding.** `/storefront/signup` was a three-field
  form with no phone, address, or verification info — not enough to
  actually deliver to or vet a new boutique. Now a 5-step wizard
  collects contact info, a structured delivery address, and an optional
  KYC field, and denormalizes phone/address onto each order so the
  dashboard shows what's needed to fulfill it. See
  [sprints/13](<sprints/[DONE] 13-guided-boutique-onboarding.md>) and
  [data-model.md](data-model.md#guided-boutique-onboarding-sprint-13).
- **Payment on the storefront** (was #6c below). `/storefront` is now
  tabbed (Catalogue / Mes commandes) and "Commander" opens a checkout
  sheet where the partner chooses mobile money (PawaPay, stubbed — no
  real credentials yet) or cash on delivery; the order→`ventes` bridge
  now writes the real paid amount instead of a hardcoded `encaisse: 0`.
  Server-side webhook confirmation is still deferred (needs Cloud
  Functions, see #10 below). See
  [sprints/08](<sprints/[DONE] 08-pawapay-payment-stub.md>) and
  [data-model.md](data-model.md#payment-sprint-08-stub-phase).
- **Admin UI for the storefront catalog** (was #6 below — this had
  already shipped in [sprints/05](<sprints/[DONE] 05-admin-catalog-management.md>)
  but this list was never updated to reflect it). Admin/staff can add,
  price, deactivate, photograph, and (sprint 14) describe a product from
  the dashboard.
- **Promo banner** (was #6b below). A single active-or-not promo, set by
  admin, shown live at the top of `/storefront`. See
  [sprints/06](<sprints/[DONE] 06-promo-banner.md>).
- **Storefront order tracking** and **self-service profile/order/product
  detail**. A partner can now see and correct their own KYC/delivery
  info at `/storefront/profile`, see a real admin-set delivery date on
  their order, and open a detail sheet for any order or product instead
  of a bare list. See [sprints/07](<sprints/[DONE] 07-storefront-order-tracking.md>)
  and [sprints/14](<sprints/[DONE] 14-storefront-self-service.md>).
- **Staff data-level scoping** (was #1 below) — **partial**. A staff
  account assigned `"Directeur de Production"` or `"Chargée de
  Commercialisation"` is now genuinely restricted at the
  `firestore.rules` level to their department's collections, not just
  hidden in the UI. Accounts with no poste (or `"Personnalisé"`) are
  unchanged — still full access, `menus` still UI-only for them. See
  [rbac.md](rbac.md#poste-based-data-scoping-sprint-17) and
  [sprints/17](<sprints/[DONE] 17-staff-poste-data-enforcement.md>).
- **Dashboard information architecture and several data/logic gaps.**
  Primes & personnel and Commercialisation (the two sections that had
  stacked 5-6 cards into one scroll) now use tabbed sub-navigation. The
  Campagne/Du/Au filter bar now actually filters on-screen data
  (previously export-only); "Stock cumulé" sums in chronological order,
  not insertion order; `charges` has an admin card (previously no UI
  at all); campaign dates in Paramètres ERP are editable; Exécutif has
  a real trend chart (`recharts` was installed but unused); Boutiques
  partenaires shows the CNI/RCCM collected at signup next to the
  "Vérifié" toggle, so admin has something to review, not a blind
  toggle. See
  [sprints/18](<sprints/[DONE] 18-dashboard-ia-data-rethink.md>).
- **Password reset**, missing entirely before. "Mot de passe oublié ?"
  on `/login`. See
  [sprints/18](<sprints/[DONE] 18-dashboard-ia-data-rethink.md>).
- **Every dashboard table row and KPI tile now opens a detail modal**
  explaining what the figure is and where it's collected, with
  edit/delete for the 11 tables backed by a real Firestore record. See
  [sprints/19](<sprints/[DONE] 19-record-detail-modals-sidebar-shortcuts.md>).
- **A "Parcours production" funnel view** connects Approvisionnement →
  Production → Stock → Commercialisation as one page, using figures
  already computed elsewhere (no new data entry) — the aggregate
  answer to "show the product's journey," distinct from literal
  per-batch traceability (roadmap #18, still not started). Every modal
  in the app is also now a centered dialog on desktop instead of a
  mobile-style bottom sheet at every width. See
  [sprints/20](<sprints/[DONE] 20-parcours-production-funnel.md>).
- **Aggregate-value modals now show a real calculation trail**, not
  just the abstract formula — Résultat brut, Marge brute, Rendement
  sur coûts, and a dozen other derived figures each show the actual
  current numbers from Approvisionnement/Production/Commercialisation
  that produced them, ending in the stated value. See
  [sprints/21](<sprints/[DONE] 21-calculation-breakdowns.md>).
- **Sidebar redundancy fixed**: Approvisionnement/Production/Stock/
  Commercialisation now group visually under "Parcours production"
  instead of standing as five near-duplicate top-level items; Parcours'
  "Voir le détail" expands inline with the clicked card highlighted
  instead of navigating away; every page's export card moved from the
  top (where it read as the primary filter) to the bottom. See
  [sprints/22](<sprints/[DONE] 22-sidebar-grouping-parcours-inline-export-move.md>).
- **Automated FC→USD conversion** on summary monetary figures (header,
  KPI tiles, Exécutif/Finances/KPI stratégiques tables) — a daily rate
  cached in `config/exchangeRate`, no manual lookup needed. "Promotion"
  also moved out of Commercialisation's tabs into its own sidebar item
  under "Parcours production," matching sprint 22's treatment of the
  other four. See
  [sprints/23](<sprints/[DONE] 23-currency-exchange-promotion-menu.md>).
- **Self-service "Mon profil"** for admin/staff, mirroring the
  storefront partner's own profile page — click the header's name/role
  to rename yourself or send a password-reset email. See
  [sprints/24](<sprints/[DONE] 24-my-profile-self-update.md>).

## Security & correctness

2. **No email verification / password reset flow** in the UI. Firebase
   Auth supports both; `signInWithEmailAndPassword` /
   `createUserWithEmailAndPassword` are wired but nothing calls
   `sendPasswordResetEmail` or `sendEmailVerification` yet.
3. **No rate limiting on `/storefront/signup`.** It's a public form backed
   directly by `createUserWithEmailAndPassword`; add App Check or a
   Cloud Function trigger if abuse becomes a concern.
4. **The generated admin password was communicated once, in plaintext,
   in this session's output.** It should be rotated on first login if it
   hasn't been already.

## Product

6d. **`clients` (internal registry) and `users` (partner accounts) are
   disconnected systems** — a storefront order never creates a `clients`
   doc, and `ventes.idClient` is free text, not a foreign key. Not
   urgent at current scale; worth knowing before it isn't.
7. **No admin UI for *managing* existing accounts** — partially closed.
   *Creating* an admin/staff account no longer needs the CLI
   (invite-link signup, see Done above), *partner* accounts have an
   admin-facing list (sprint 16's "Boutiques partenaires" card — name,
   contact, address, a verification toggle), and *staff* accounts have
   one too (sprint 17's "Équipe (staff)" card — name, email, poste
   assignment). Still CLI-only: listing all fields, deactivating, or
   changing the *role* of an admin/staff account
   (`AROM-Backend/scripts/list-users.mjs` + a manual Firestore console
   edit) — neither card covers those.
8. **`reset()` in the ERP store is now a no-op** (with a toast pointing to
   admin scripts) rather than actually clearing data — intentional, since
   the old "reset to seed" behavior would have wiped shared, live
   Firestore data for every user with one click. If a real "start a new
   campaign" flow is wanted, it should be a deliberate admin action
   (e.g. an Admin SDK script that archives the current campaign under a
   dated collection rather than deleting it).

## Infrastructure

9. **Budget alert.** Billing is linked to `arom-production` but no budget
   alert is configured. Set one in the GCP console
   (Billing → Budgets & alerts) — cheap insurance given usage should be
   ~$0 at this scale.
10. **Cloud Functions** would let things that genuinely need server
    compute — welcome/notification emails, the PawaPay webhook upgrade
    (see [flows.md](flows.md#architecture)) — happen without a manually-run
    script. (Account provisioning turned out *not* to need this — see
    invite-link signup in Done above.) Deliberately deferred since it
    wasn't needed for a working v1 and keeps the project off any
    Functions-related billing surface until it's actually wanted.
11. **Nothing currently deploys `AROM-Production`.** Was entirely owned by
    Lovable's pipeline; the project has been disconnected from Lovable
    (2026-08-14, see [architecture.md](architecture.md#update-2026-08-14-lovable-removed)).
    An independent Cloudflare Workers pipeline is now built (sprint 12,
    [Teddmab/AROM-Production#11](https://github.com/Teddmab/AROM-Production/pull/11))
    but not yet live — it needs `CLOUDFLARE_API_TOKEN` and
    `CLOUDFLARE_ACCOUNT_ID` as GitHub repo secrets before pushes to
    `main` actually deploy. See
    [runbook.md](runbook.md#frontend-deploys) for exactly what's needed.

## Backlog toward viable V1

Surfaced by a 2026-08-15 audit alongside [sprints/18](<sprints/[DONE] 18-dashboard-ia-data-rethink.md>)
(dashboard IA/data rework) — real gaps, deliberately not bundled into
that sprint because each is bigger scope on its own. In rough priority
order:

12. **Storefront inventory / stock-out prevention.** `products` has no
    `stock` field at all — the storefront quantity stepper only floors
    at 0, with no upper bound tied to `computed.stockPF` (the real
    finished-goods count, which lives only in the internal ERP Stock
    tab, completely disconnected from the storefront catalog). A
    partner can order more bottles than physically exist. Needs a
    `stock` field on `products`, a way to set it from the Catalogue
    card, and a checkout-time validation — real operational risk at
    production scale.
13. **Full admin/staff account management UI.** Deactivating an
    account or changing its role is still CLI-only
    (`scripts/list-users.mjs` + a manual Firestore console edit).
    Sprint 16/17's "Boutiques partenaires"/"Équipe" cards cover
    verification and poste assignment, not this. Same gap as roadmap
    #7 above, restated here because it's part of the same "what's
    missing for V1" audit.
14. **New-boutique-signup notification.** Sprint 18 added an
    unverified-first sort to Boutiques partenaires so new signups
    aren't buried alphabetically, but there's still no push/email
    alert when one signs up — admin only finds out by opening the
    dashboard.
15. **`recharts` beyond the one Exécutif trend chart** (sprint 18).
    Commercialisation, Finances, and Stock all still present their
    numbers as tables only; each could get a real chart the same way
    Exécutif just did.
16. **Table pagination/search.** Every table in the dashboard (ventes,
    productions, invites, …) renders its full unfiltered array with no
    paging or search. Not urgent at current scale (a few dozen rows
    per collection) but will need addressing as a campaign's data
    grows.
17. **Email verification.** `sendEmailVerification` is unused —
    nothing confirms a signup's e-mail address is real, for any role.
18. **Literal batch traceability** — still not started, distinct from
    the item below. Raised alongside sprint 19: group Approvisionnement,
    Production, Stock, and Commercialisation under one place and let
    admin see one physical batch's path from raw fruit receipt through
    to sale. The current data model has no linking fields between
    those four collections (a `Production` doc doesn't reference which
    `Approvisionnement` receipt(s) it consumed, `stockMP` doesn't
    reference the production lot that generated it, etc.), so this
    needs a real data-model design pass — deciding what a "batch" is
    and how it's threaded through four collections that were each
    designed independently — before any UI grouping is meaningful.
    [Sprint 20](<sprints/[DONE] 20-parcours-production-funnel.md>)
    answered the underlying "journey" ask a different way (see below)
    without starting this.

## From the 2026-08-14 kickoff meeting (client: Ethical Mine)

Client walkthrough of the current build. Two things confirmed as
already satisfying what was asked for, two new asks captured as
sprints, and a longer-term vision worth recording so it isn't lost.

**Already satisfied by shipped/in-review work:**
- "Améliorer commandes" (better order management, visible delivery
  status/date) — this is exactly [sprints/07](<sprints/[DONE] 07-storefront-order-tracking.md>)
  and [sprints/14](<sprints/[DONE] 14-storefront-self-service.md>)'s order detail
  sheet, both shipped.
- Google/Facebook login, new boutiques landing on the catalog after
  signup, separate catalog/orders views — all already shipped
  (sprint 10, and the catalogue/orders tabs work).
- "Implémenter KYC" (a simple, human-in-the-loop verification for new
  boutiques) — [sprints/16](<sprints/[DONE] 16-boutique-verification.md>),
  distinct from and in addition to the KYC *data collection* sprint 13
  already shipped; also the first real (partial) answer to #7 below.

**New ask, still open:**
- [sprints/15](sprints/15-whatsapp-notifications.md) — WhatsApp
  notifications on order confirm/fulfill. Needs a WhatsApp Business API
  or Twilio-style provider decision before it can be more than a stub —
  see the sprint file's "Décisions déjà actées."

**"Mise en ligne" (go live)** is sprint 12 — the Cloudflare Workers
pipeline is built and waiting on the client to provide
`CLOUDFLARE_API_TOKEN`/`CLOUDFLARE_ACCOUNT_ID`. Worth flagging back to
the client directly since they asked for this as a next step and it's
blocked on them, not on build work.

**Longer-term vision (explicitly "nice to have," not scheduled):**
- **Approvisionnement → Mombongo integration.** Link AROM's supply data
  to a Mombongo administrator account so the ~25 partner
  plantations' data syncs automatically instead of manual entry, with
  an "import" counterpart to the existing ERP export (import from
  Excel, from Mombongo, or from Agroconnect). The client was explicit
  this depends on Mombongo's own maturity and AROM's own process
  simplification first — "pas pour tout de suite."
- **Auto-populate production from approvisionnement.** Today both are
  filled by hand; quantities already entered at the approvisionnement
  step could pre-fill the matching production entry. Mentioned as
  something already being investigated (data-schema study), not yet a
  committed design.
- **Agroconnect integration**, positioned as helping other juice
  producers ("transformateurs") source raw pineapples, not something
  AROM itself needs directly.
- **Geographic expansion** (Kinshasa, Lubumbashi, then possibly Rwanda,
  Tanzania) — the underlying reason the Mombongo/Agroconnect
  integrations matter at all: the same approvisionnement flow should
  work for a new AROM location finding new suppliers in a new region,
  not just Kasaï. No scope or timeline attached yet.
