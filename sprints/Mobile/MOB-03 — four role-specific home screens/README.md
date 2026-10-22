# MOB-03 — Role-Specific Home Screens

> **Status: Done** (verified 2026-09-01). `RoleHomeScreen.tsx` +
> `roleHomeConfig.ts`/`roleHomeData.ts` drive one reusable shell across
> COLLECTION_AGENT/PRODUCTION/SALES; ADMIN was later forked into its own
> screen set by MOB-06A (a spec-compliant evolution, not a regression).
> Covered by `RoleHomeScreen.test.tsx`, `HomeScreen.test.tsx`,
> `BottomNav.test.tsx`, `AppHeader.test.tsx`. Two `comingSoon` actions
> (stock movement, new sale) were left as honest stubs per this sprint's
> own "no fake form" rule — both replaced with real flows in MOB-11 (see
> `MOB-11 — MVP completion (...)`), so no `comingSoon` role-home action
> remains anywhere in the app as of 2026-09-01.

## Objective

After MOB-02 completes, route the user to a useful home screen tailored to the active authorized role. Implement one reusable home shell with four configurations, not four duplicated applications.

## Roles and home priorities

### 1. Administrateur

Purpose: see the whole operation at a glance and identify what needs attention.

- Campaign and synchronization status
- Today overview: collections, production, stock alerts and sales
- Pending approvals/alerts
- Shortcuts to operational, commercial and financial follow-up
- Next priority card

### 2. Agent de collecte

Purpose: know where to go and record the next reception quickly.

- Today's planned collections
- Next producer/visit
- Collections completed and pending synchronization
- Primary action: `Nouvelle réception`
- Next task card

### 3. Production & stock

Purpose: start the next batch and protect stock accuracy.

- Production queue and lots awaiting quality control
- Finished products available
- Low-stock or discrepancy alerts
- Primary actions: `Lancer une production` and `Mouvement de stock`
- Next task card

If production and stock are separate backend roles, compose this home from the permissions actually granted. Do not widen either role.

### 4. Ventes & boutique

Purpose: process orders and record payments quickly.

- Today's sales and cash received
- Orders awaiting preparation
- Customer balances requiring follow-up
- Primary actions: `Nouvelle vente` and `Commandes`
- Next order/task card

## Shared home shell

Every home must include:

- AROM header, greeting and notification affordance
- Persistent active-role chip and safe `Changer de rôle` access
- Active campaign selector/readout
- Synchronization status with last-sync time
- Role-specific realistic hero image
- Maximum four concise metrics
- Two to four large role-authorized actions
- `Aujourd’hui` summary
- One `Prochaine tâche` or `À suivre` card
- Pull to refresh
- Shared bottom navigation: `Accueil`, `Tâches`, central `+`, `Activité`, `Plus`

The central `+` must expose only already-implemented and authorized actions. If MOB-04 owns an action not yet implemented, do not create a fake form or broken route; hide that action until available.

## Data and authorization

- Read from existing Firestore collections, repositories and cached MOB-02 data.
- Reuse server-authorized role and poste visibility.
- Never invent collections, fields, totals or endpoints.
- Calculated values must use existing domain rules; otherwise display only confirmed raw counts.
- Scope all queries by authenticated uid, organization and active role/permissions.
- Keep actions permission-aware even if a user reaches a route by deep link.
- The known task gap remains: tasks are poste-scoped, not mobile-role scoped. Reuse poste visibility and document the limitation; do not invent an assignee field.

## Offline and system states

Each home must provide:

- Skeleton loading
- Empty state with an appropriate next action
- Offline banner using the last synchronized snapshot
- Pending/failed synchronization indicator
- Retry action where supported
- Partial-card failure without blanking the whole dashboard
- Unauthorized-role and expired-session handling

Never present cached metrics as live without an offline/last-updated label.

## Design rules

Follow the attached reference and the established AROM system:

- Cream `#FBF7EB`, deep green `#005A2B`, yellow `#F2B632`
- French, short and reassuring copy
- Large cards, 48px targets, clear icon + text
- Realistic Congolese/African operational photography
- No dense tables or horizontally scrolling KPI strips
- Support small Android phones, iOS safe areas and large text
- Red only for destructive actions or critical errors

## Architecture

Prefer reusable pieces such as `RoleHomeScreen`, `HomeHeader`, `RoleBadge`, `CampaignStatus`, `SyncStatus`, `PhotoActionCard`, `TodayMetric`, `NextTaskCard`, `OfflineBanner` and a typed role configuration. Reuse existing equivalents before adding new components.

## Acceptance criteria

- MOB-02 ready action reaches the correct role home.
- All four authorized role configurations render correctly.
- Switching roles updates queries, actions and labels without leaking prior-role data.
- Administrator sees the four operational areas in one view.
- Production/stock permissions remain distinct when the backend separates them.
- Bottom navigation is consistent and protected.
- Loading, empty, offline, partial-error and unauthorized states are tested.
- No placeholder business figures are shipped.
- Relevant unit/component/navigation tests pass together with the existing 96-test baseline.

## Out of scope

New reception, production, stock movement and sale forms; full quick-action sheet; notification centre; activity timeline; sync centre; management reports. These belong to later sprints.

