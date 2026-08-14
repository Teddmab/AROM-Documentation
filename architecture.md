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
  unrelated to auth. Removed 2026-08-14 along with the rest of the
  Lovable dependency, see "Update" below.

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
  typecheck, build on every push/PR to `main`. Does not deploy — see the
  2026-08-14 update below for what changed here.
- **`AROM-Backend/.github/workflows/deploy-rules.yml`** — deploys
  `firestore.rules`, `firestore.indexes.json`, `storage.rules` to
  `arom-production` on every push to `main` touching those files, using
  the `FIREBASE_SERVICE_ACCOUNT_KEY` repo secret. Unaffected by the
  Lovable removal — this pipeline never depended on Lovable.

## Topology (as of 2026-08-14)

```
                 ┌─────────────────────────┐
   visitors ───► │  Cloudflare Workers      │  AROM-Production (frontend)
                 │  (SSR + static)          │  — deploy pipeline: see the
                 └─────────────┬────────────┘    2026-08-14 update below
                                │ Firebase Web SDK
                                ▼
                 ┌─────────────────────────┐
                 │   Firebase: arom-production
                 │   - Auth (email/password, Google, Facebook)
                 │   - Firestore (ERP data, users, invites, products, orders)
                 │   - Storage (product photos)
                 └─────────────┬────────────┘
                                │ rules deploy (CI) / admin scripts
                                ▼
                 ┌─────────────────────────┐
                 │   AROM-Backend           │  rules, seed & account scripts
                 └─────────────────────────┘
```

## Update (2026-08-14): Lovable removed

`AROM-Production` was disconnected from Lovable — both the GitHub
integration (on Lovable's own dashboard, outside this repo) and every
piece of Lovable-specific code:

- `vite.config.ts` no longer depends on `@lovable.dev/vite-tanstack-config`.
  Read the wrapper's actual installed source rather than reverse-engineering
  it from behavior, and rebuilt the same config directly: TanStack Start,
  `@tailwindcss/vite`, `vite-tsconfig-paths`, Nitro with the
  `cloudflare-module` preset (build only, not during `vite dev`),
  `@vitejs/plugin-react`, the same path alias/dedupe list. Every package
  needed was already a direct dependency — nothing new to install. Dropped
  intentionally: Lovable's dev-only error-diagnostics plugins and the
  sandbox-only asset proxy/HMR-gate/dev-server-bridge plugins, all
  confirmed no-ops outside Lovable's own sandbox environment by reading
  the source, not assumed safe to drop.
- The MCP tool server (`@lovable.dev/mcp-js`, `src/lib/mcp/`, the `/mcp`
  and `[.mcp]` routes) is gone — a deliberate scope decision made when
  disconnecting, not a side effect. Nothing else depended on it.
- `src/lib/lovable-error-reporting.ts`, `.lovable/`, and `AGENTS.md`
  (which only ever contained the Lovable connection notice) are gone.
- Two asset URLs that only resolved through Lovable's own hosting
  (`arom-hero.asset.json`'s `/__l5e/assets-v1/...` path, and a
  `gpt-engineer-file-uploads` GCS-hosted social image) were 404ing in
  any non-Lovable environment — including local dev, which is how this
  got caught. Both now point at `/logo-hero.png`.

**Consequence that matters operationally**: nothing currently deploys
`AROM-Production`. Lovable's pipeline was the only thing that did.
Setting up an independent Cloudflare Workers deploy pipeline (a
`wrangler` config + GitHub Actions, needing a Cloudflare API token and
account ID) is a deliberate, separate follow-up — not bundled into the
removal itself, and not yet done as of this update. See
[runbook.md](runbook.md#frontend-deploys) for current status.

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
