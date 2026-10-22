# MOB-06A — confirmed research (recorded before implementation)

Answers to the "Mandatory research before implementation" list in README and the
Phase 1 evidence report in IMPLEMENTATION_PROMPT.md. Confirmed by reading
AROM-Mobile, AROM-Backend, AROM-Production and AROM-Documentation directly on
2026-08-29. Facts only; see "Amendments" section in README.md for the scope
changes these findings require.

1. **producteurs** — `producteurs/{id}`: `id, nom, village, secteur, territoire,
   telephone, produit, capaciteKgMois, prixConvenu, statut, observations,
   photoUrl?, createdByUid?, createdByPoste?, createdAt?`. Type:
   `AROM-Mobile/src/features/reception/model.ts:17-46`, mirrored
   `AROM-Production/src/lib/erp/model.ts:37-65`, documented
   `AROM-Documentation/data-model.md:17`, validated
   `AROM-Backend/firestore.rules:172-186`. **No org-scoping field exists
   anywhere in the system** — see item 10.

2. **Receptions (`approvisionnements/{id}`)** — has quantity (`qteRecueKg`),
   agreed unit price (`prixKg`) and cost fields (`transport`, `autresFrais`):
   `AROM-Production/src/lib/erp/model.ts:67-88`, rules
   `firestore.rules:188-203`. **No stored total/payable field.** Authoritative
   formula already exists — `calcAppro` in
   `AROM-Production/src/lib/erp/engine.ts:20-22`:
   `valeurAchat = qteRecueKg * prixKg; coutTotal = valeurAchat + transport + autresFrais`.
   Any "estimated amount due" in MOB-06A must reuse this exact formula, not
   reimplement it.

3. **productions** — `productions/{id}`: `id, lot, date, kgUtilises,
   volumeJusL, q500, q330, q300, rejets, responsable, staffUid?, statut,
   approvisionnementIds?`. `AROM-Production/src/lib/erp/model.ts:90-110`,
   rules `firestore.rules:206-217`.

4. **qualityControls** — `qualityControls/{id}`, type
   `AROM-Mobile/src/features/quality/model.ts:22-45`. Rules validate shape
   only (`firestore.rules:229-246`); **the one-control-per-lot rule is
   enforced client-side in a transaction**, not by rules —
   `AROM-Mobile/src/features/quality/qualitySync.ts:149-196`. Offline
   dependency chaining already exists: a control queued against an unsynced
   production is enqueued with `dependsOn = productionId`
   (`qualitySync.ts:92-107`); `syncPendingQualityControls`
   (`qualitySync.ts:236-263`) treats a control blocked on its production as
   "blocked, not failed" and skips it rather than surfacing it as absent.
   **This reconciliation primitive already exists and must be reused as-is**
   for A3/A4's "lot awaiting control vs. locally queued control" distinction
   — do not reinvent it.

5. **stockMP** — `stockMP/{id}` is a movement ledger (`type: Entrée|Sortie|
   Ajustement`, not a running balance):
   `AROM-Production/src/lib/erp/model.ts:112-133`,
   `AROM-Documentation/data-model.md:21`. Confirmed raw-material-only — no
   finished-goods stock collection exists anywhere; finished-goods stock
   (`stockPF`) exists only as an in-memory computed value in the web ERP
   (`AROM-Production/src/lib/erp/engine.ts:85,160-245`), never a Firestore
   document. **Stock threshold: confirmed absent** across all four repos —
   do not invent one, per README's own instruction.

6. **sales (`ventes/{id}`)** — `numero, date, idClient, client, canal,
   format, quantite, prixUnitaire, remise, encaisse, commerciale, staffUid?,
   productionIds?` (`AROM-Production/src/lib/erp/model.ts:145-167`,
   `AROM-Documentation/data-model.md:23`). Computed fields (`montantBrut`,
   `soldeDu`, `statutPaiement`) live only in
   `AROM-Production/src/lib/erp/engine.ts:50-62`, never stored. AROM-Mobile
   already reads `ventes` for ADMIN/SALES home metrics
   (`AROM-Mobile/src/features/home/roleHomeData.ts:51,77`); no mobile
   sales-entry UI exists yet.

7. **tasks** — `tasks/{id}`: `id, stage, title, sourceLabel, sourceId?,
   status (pending|done), createdAt, completedAt?, completedBy?`
   (`AROM-Production/src/lib/erp/tasks.ts:26-38`). Rules:
   `read/update/delete: isAdminOrStaff()` (`firestore.rules:344-355`).
   **No per-account assignee field exists.** `AROM-Mobile/src/features/sync/
   syncPlan.ts:27-36` already documents this: every admin/staff account sees
   the full pending-task list, not one scoped to them. See Amendment 3.

8. **users/staff** — `users/{uid}`: `email, displayName, role, active,
   poste?, menus, createdAt` (`AROM-Documentation/data-model.md:35`, mobile
   subset `AROM-Mobile/src/lib/firebase/auth.tsx:32-39`).
   **`lastActivity`/last-seen: confirmed absent** anywhere in schema, rules
   or mobile code. A5 "staff activity" must derive only from real business
   events (`qualityControls.staffUid`/`responsable`, `productions.staffUid`,
   `approvisionnements`), never from a login/presence signal — README
   already anticipates this correctly.

9. **Invoice/payment/Mombongo** — no `invoice`/`facture` concept exists
   anywhere. The only real payment plumbing is `orders/{id}.payment`
   (`AROM-Documentation/data-model.md:38`, PawaPay mobile-money for
   storefront customer orders — `AROM-Production/src/lib/payments/
   pawapay.ts`), which is the **downstream customer payment path, unrelated
   to upstream producer payables**. Mombongo appears only in
   `AROM-Documentation/roadmap.md:271-276` as an unbuilt future integration
   — the "no Mombongo branding" rule is protecting against something that
   doesn't exist in code yet, not removing a live dependency.

10. **ADMIN rules / organization isolation** — `isAdmin()`
    (`firestore.rules:18-20`) already has full read/write on every
    collection MOB-06A touches. **Organization isolation is confirmed
    absent system-wide** — there is no `orgId`/`organisationId` field or
    concept anywhere in AROM-Backend, AROM-Production, AROM-Mobile or
    AROM-Documentation. See Amendment 1.

11. **App shell** — `AppHeader` (`AROM-Mobile/src/features/home/components/
    AppHeader.tsx:26-61`) is the single shared header, rendered by
    `AppShell`. **`BackHeader` is dead** — `AppHeader.tsx:11` states it
    "supersedes BackHeader.tsx"; the component no longer exists in the tree.
    See Amendment 2. `BottomNav` already renders exactly `Accueil · Tâches ·
    + · Activité · Plus` — no change needed. Current ADMIN home is the
    generic `RoleHomeScreen` (`RoleHomeScreen.tsx:20-120`), same component
    every role uses, configured only by data tables
    (`roleHomeConfig.ts`, `roleHomeData.ts`). **ADMIN's cache-first sync
    plan does not currently include `qualityControls` or `stockMP`** —
    `AROM-Mobile/src/features/sync/syncPlan.ts:121-144` only pulls
    `campagne, equipe, instructions, approvisionnements, productions,
    ventes` for ADMIN. Both `getSyncPlan` and `CATEGORY_KEYS_BY_ROLE` must
    be extended before A1/A3 can honestly claim cache-first quality/stock
    data. `/activity` (`AROM-Mobile/src/app/activity.tsx`) is an explicit
    stub with no real feed — A2 must be built from scratch reading
    `approvisionnements`/`productions`/`ventes`/`qualityControls` directly.
    `/team` is a working read-only roster off the `equipe` cache category —
    reusable for A5.

12. **Shared role-home architecture** — all four roles render the identical
    `RoleHomeScreen` shell, differing only by config tables. MOB-06A's
    dashboard is six distinct sub-screens (A1–A6), materially larger than
    what this shared generic shell was built for. See Amendment 4.

Most load-bearing files for implementation: `AROM-Production/src/lib/erp/
engine.ts` (authoritative money math), `AROM-Mobile/src/features/quality/
qualitySync.ts` (dependency/conflict pattern to reuse for invoice sync),
`AROM-Mobile/src/features/sync/mutationQueue.ts` + `syncPlan.ts` (extend for
ADMIN), `AROM-Mobile/src/features/home/roleHomeConfig.ts` +
`roleHomeData.ts` (ADMIN's config to replace/extend),
`AROM-Backend/firestore.rules:18-64` (role/poste predicates — no org
concept), `AROM-Documentation/data-model.md:17-38`.
