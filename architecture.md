# Architecture — deep analysis

## What AROM is

AROM is a small pineapple-juice production unit in the Kasaï region (DRC).
The app in `AROM-Production` is a transposition of an Excel workbook
(`AROM_ERP_Professionnel.xlsm`) into a web ERP: it tracks a monthly
"campagne" from raw-fruit purchase through production, stock, sales,
marketing, finance, and staff bonuses, plus a marketing landing page and
(new, as of this pass) a partner storefront.

## State before this work (2026-08-13)

The app was a **Lovable-generated prototype**, functionally a static
single-tenant demo:

- **Stack**: TanStack Start (React 19, file-based routing) + Vite, styled
  with Tailwind v4, built to a Cloudflare Workers target via Nitro
  (`vite.config.ts` locks this in through `@lovable.dev/vite-tanstack-config`
  — Lovable's own pipeline deploys the app; nothing else does).
- **No git repository.** The project had never been committed or pushed
  anywhere outside Lovable's own project store.
- **No backend.** `src/lib/erp/store.tsx` held all ERP data (`producteurs`,
  `approvisionnements`, `productions`, `stockMP`, `clients`, `ventes`,
  `marketing`, `charges`, `parametres`) in a React context backed by
  `localStorage`. The header even displayed a green **"ERP connecté"**
  badge while every write only ever touched the visiting browser's local
  storage — nothing was shared between users or devices, and clearing
  browser storage discarded the campaign.
- **No authentication, no access control.** `/dashboard` — the "tableau de
  bord" covering all 11 ERP modules, including finances and primes — was a
  plain public route. Anyone with the URL had full read/write access to
  every module, with no concept of a user at all.
- **No commercial storefront.** There was no way for an external buyer
  ("partenaire") to see the catalog or place an order; `ventes` entries
  could only be typed in by whoever had the dashboard open.
- **An MCP tool server was already wired up** (`src/lib/mcp/`,
  `src/routes/mcp.ts`) exposing read-only ERP summaries as MCP tools —
  unrelated to auth, but worth knowing it's there.

In short: a working UI and a correct calculation engine
(`src/lib/erp/engine.ts`), sitting on zero persistence and zero access
control.

## What changed in this pass

### 1. Backend: Firebase

A new Firebase project, **`arom-production`**, provides:

- **Auth** — email/password sign-in (Identity Platform-backed; required
  linking a GCP billing account even for free-tier usage — see
  [runbook.md](runbook.md) if this needs to be redone for a new project).
- **Firestore** (Native mode, `eur3` multi-region) — one document per ERP
  row, one collection per module, plus `products` and `orders` for the
  storefront. See [data-model.md](data-model.md).
- **Storage** (`arom-production.firebasestorage.app`) — reserved for
  product photos; not yet used by the UI.

Config lives in `Teddmab/AROM-Backend`: `firestore.rules`,
`storage.rules`, `firestore.indexes.json`, plus Admin-SDK scripts to
provision accounts and seed data. A dedicated service account
(`arom-ci-deploy@arom-production.iam.gserviceaccount.com`, `roles/firebase.admin`)
deploys rule changes via GitHub Actions on every push to `main`.

### 2. Frontend: real persistence

`src/lib/erp/store.tsx` was rewritten to back the same public API
(`useErp()` → `state`, `computed`, `addRow`, `removeRow`,
`updateParametres`) with Firestore `onSnapshot` listeners and
`setDoc`/`deleteDoc`/`updateDoc` writes instead of `localStorage`. No
section component (`ExecutiveSection`, `ApproSection`, etc.) needed to
change — they only ever consumed the hook's public shape. Data entered in
the dashboard is now real-time, shared, and durable.

### 3. Access control

- `src/lib/firebase/auth.tsx` — `AuthProvider`/`useAuth()`, mirroring
  Firebase Auth state to a live `users/{uid}` Firestore doc (`role`,
  `menus`, `active`).
- `src/lib/firebase/require-role.tsx` — `<RequireRole roles={[...]}>`
  gate: redirects to `/login` (unauthenticated) or `/storefront`
  (authenticated but wrong role) while auth state resolves client-side.
- `/dashboard` is now wrapped in `<RequireRole roles={["admin","staff"]}>`.
  The sidebar/section-select only lists sections a `staff` account's
  `menus` field grants (`admin` always sees everything) — see
  [rbac.md](rbac.md).
- `/login` — email/password sign-in for admin/staff/partner.

### 4. Partner storefront

- `/storefront/signup` — self-registration; the only role a client-side
  write is allowed to create for itself (see `firestore.rules`).
- `/storefront` — product catalog (`products` collection), cart, order
  placement (`orders` collection scoped to the signed-in partner), and
  order history.

### 5. CI/CD

Two pipelines, deliberately scoped to what each repo actually owns:

- **`AROM-Production/.github/workflows/ci.yml`** — install, lint,
  typecheck, build on every push/PR to `main`. It does **not** deploy —
  Lovable's own pipeline already builds and deploys this app to Cloudflare
  Workers on every push to the connected branch, and this repo's CI has no
  Cloudflare credentials. Duplicating that deploy here would either fight
  Lovable's pipeline or require credentials this session doesn't have; the
  right move if that's ever wanted is a deliberate follow-up, not a
  silent addition.
- **`AROM-Backend/.github/workflows/deploy-rules.yml`** — deploys
  `firestore.rules`, `firestore.indexes.json`, `storage.rules` to
  `arom-production` on every push to `main` touching those files, using
  the `FIREBASE_SERVICE_ACCOUNT_KEY` repo secret.

## Topology

```
                 ┌─────────────────────────┐
   visitors ───► │  Lovable → Cloudflare    │  AROM-Production (frontend)
                 │  Workers (SSR + static)  │  — builds/deploys on push,
                 └─────────────┬────────────┘    independent of GitHub CI
                                │ Firebase Web SDK
                                ▼
                 ┌─────────────────────────┐
                 │   Firebase: arom-production
                 │   - Auth (email/password)
                 │   - Firestore (ERP data, users, products, orders)
                 │   - Storage (product photos)
                 └─────────────┬────────────┘
                                │ rules deploy (CI) / admin scripts
                                ▼
                 ┌─────────────────────────┐
                 │   AROM-Backend           │  rules, seed & account scripts
                 └─────────────────────────┘
```

## Deliberate non-goals of this pass

- **No Cloud Functions.** Everything (role checks, order writes) is done
  with Firestore security rules and client-side Admin-SDK scripts, so the
  project stays fully usable on Firebase's Spark-equivalent usage even
  though billing is now linked (billing was required to enable Auth
  itself, not to use Functions). Custom claims / Cloud Functions are a
  reasonable future upgrade if the account-provisioning workflow needs to
  move off manual scripts — see [roadmap.md](roadmap.md).
- **No payment integration.** Storefront orders are a request/fulfillment
  workflow (`pending → confirmed → fulfilled`), not a checkout — AROM
  staff confirm and fulfill manually for now.
- **No deploy step for the frontend in CI.** See CI/CD above.
