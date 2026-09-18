# Runbook

## Local development

```
git clone https://github.com/Teddmab/AROM-Production.git
cd AROM-Production
bun install
bun run dev
```

No `.env` needed for local dev against the live `arom-production` project
— `src/lib/firebase/config.ts` ships working defaults. Copy `.env.example`
to `.env.local` only if pointing at a different Firebase project.

## Local development against the Firebase Emulator Suite

For changes that write data (anything touching Firestore/Auth), prefer
testing against the emulator suite over live `arom-production` data.
`AROM-Backend/firebase.json` already has emulator ports configured
(Auth 9099, Firestore 8080, Storage 9199, UI 4000).

```
# Terminal 1 — emulators (from AROM-Backend)
firebase emulators:start --project demo-arom-local
# UI at http://127.0.0.1:4000 — if port 8080 is already taken locally,
# bump "emulators.firestore.port" in firebase.json for the session.

# Terminal 2 — seed data + a test admin account (from AROM-Backend)
FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 FIREBASE_AUTH_EMULATOR_HOST=127.0.0.1:9099 \
  GCLOUD_PROJECT=demo-arom-local npm run seed
FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 FIREBASE_AUTH_EMULATOR_HOST=127.0.0.1:9099 \
  GCLOUD_PROJECT=demo-arom-local npm run create-user -- admin@test.local "TestPass123!" admin "Test Admin"

# Terminal 3 — frontend (from AROM-Production), .env.local:
#   VITE_USE_FIREBASE_EMULATOR=true
#   VITE_FIREBASE_PROJECT_ID=demo-arom-local
#   VITE_FIREBASE_EMULATOR_FIRESTORE_PORT=8080   # only if you changed the port above
bun run dev
```

Partner accounts self-register normally at `/storefront/signup` — no
script needed, same as against live data. Emulator data is in-memory and
resets when the emulator process stops (`--import`/`--export-on-exit`
flags exist if persistence across restarts is ever wanted).

**Seeing every emulator error, not just what the CLI prints.** The
Firestore and Storage emulators are separate JVM processes; the Firebase
CLI only streams a subset of their output to Terminal 1 above. Full
per-request detail — including the exact rule line a `PERMISSION_DENIED`
failed at — is written to `firestore-debug.log` / `storage-debug.log` in
`AROM-Backend`'s working directory instead, silently, even when nothing
prints to the terminal. Tail those live in another terminal while the
emulators run:

```
# Terminal 4 (from AROM-Backend)
tail -f firestore-debug.log storage-debug.log
```

`--log-verbosity DEBUG` on `emulators:start` does not change this — it
only affects the Firebase CLI's own lifecycle logging, not where the
Firestore/Storage JVM processes write their request-level errors.

## Creating an admin or staff account

```
git clone https://github.com/Teddmab/AROM-Backend.git
cd AROM-Backend
npm install
```

Get a service account key once (Firebase console → Project settings →
Service accounts → Generate new private key for `arom-production`), keep
it outside the repo, then:

```
GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json \
  npm run create-user -- person@example.com "a-strong-password" admin "Full Name"

GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json \
  npm run create-user -- staff@example.com "a-strong-password" staff "Full Name" appro,production,stock
```

Re-running with the same email updates the existing account (password,
name, role, menus) instead of erroring.

List everyone with a role doc:

```
GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json npm run list-users
```

## Partner accounts

No script needed — partners self-register at `/storefront/signup` on the
live app.

## Re-seeding campaign data

`npm run seed` (with `GOOGLE_APPLICATION_CREDENTIALS` set) writes the
campagne pilote 2026 baseline into Firestore with `merge: true` — safe to
re-run, it won't duplicate rows (same IDs), but it **will** overwrite
fields on existing docs with the seed's values. Don't run it against a
project with real campaign data you don't want reverted.

## Deploying security rule changes

Normally automatic: edit `firestore.rules` / `storage.rules` /
`firestore.indexes.json` in `AROM-Backend`, push to `main`, and
`.github/workflows/deploy-rules.yml` deploys them.

To deploy by hand:

```
GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json \
  npx firebase deploy --only firestore:rules,firestore:indexes,storage --project arom-production
```

## Rotating the CI service account key

```
gcloud iam service-accounts keys create key.json \
  --iam-account=arom-ci-deploy@arom-production.iam.gserviceaccount.com
gh secret set FIREBASE_SERVICE_ACCOUNT_KEY --repo Teddmab/AROM-Backend < key.json
rm key.json
```

List and revoke old keys with `gcloud iam service-accounts keys list
--iam-account=arom-ci-deploy@arom-production.iam.gserviceaccount.com`.

## Inventory-service system account (Sprint 08, Step B)

`AROM-Production`'s `/api/inventory/qc-release` route signs in as a
dedicated, narrow-privilege Firebase Auth account
(`inventory-service@system.arom.cd`) to write `stockPF`/`stockBalance`/
`stockLotBalance` — the only identity `firestore.rules`'
`isInventoryService()` accepts for those writes. It carries exactly one
custom claim (`inventoryService: true`) and a `users/{uid}` doc with
`role: "system"` — never `admin`, never any `poste`, so it can never pass
`isAdmin()`/`isAdminOrStaff()`/any staff-poste check anywhere else in the
rules file. Same pattern as the pre-existing Mombongo webhook account
(`mombongo-webhook@system.arom.cd`) — a separate identity per narrow
purpose, never shared across domains.

**Provisioning** — rewritten 2026-09-18 (deployment-safety hardening)
after an audit found the prior version printed the generated password to
stdout, exposing a live production credential to any terminal/log/agent
transcript that captured it. The password is now never printed, logged,
or written to any file — it's generated in memory and piped straight
into `wrangler secret put`'s stdin, which this script now does for you:

```
GOOGLE_APPLICATION_CREDENTIALS=/path/to/key.json \
  node scripts/provision-inventory-service-account.mjs \
    --project arom-production-657f2 --worker arom-production
```

Both `--project` and `--worker` (or `INVENTORY_SERVICE_TARGET_PROJECT`/
`INVENTORY_SERVICE_TARGET_WORKER` env vars) are required for any
non-emulator run and must equal the one real project/Worker literally —
never a default, never inferred. Running under `firebase emulators:exec`
(any `--project`) targets the local emulator instead and skips the
project check, matching how every other script in this repo distinguishes
emulator from live runs (`scripts/lib/admin.mjs`).

Add `--dry-run` first to see exactly what a real run would do (does the
account already exist? which of the 3 Worker secret names already exist?
is `wrangler` authenticated?) without creating an account, generating a
password, setting a claim, or writing a secret.

**An account that already exists is never modified** — this script only
ever creates an absent one (a change from the prior "self-correcting"
re-run behavior, deliberately: automatically touching an existing
production identity, even to "just" re-apply its claim, is exactly the
kind of automatic mutation this hardening removes). If the account
exists and its claim or profile ever needs correcting, do that
explicitly and manually via the Firebase console or a one-off Admin SDK
call — never by re-running this script.

On success, all 3 Worker secrets are set automatically by the script
itself — `INVENTORY_SERVICE_EMAIL`, `INVENTORY_SERVICE_PASSWORD`, and
`FIREBASE_WEB_API_KEY` (only if `FIREBASE_WEB_API_KEY_VALUE` is set in
your own shell's environment first — see below), each piped to
`wrangler secret put <NAME> --name arom-production` via stdin, never as
a command argument. Nothing further to run manually unless that report
shows a `skipped-manual` or `failed` step.

**The Firebase Web API key** identifies this project for the Identity
Toolkit sign-in call the Worker makes as this account — it's a
non-secret, publicly-embedded project identifier by design (the exact
same value is already hardcoded directly in `AROM-Mobile`'s own committed
`src/lib/firebase/config.ts`), but the script still never fabricates or
prints a value you haven't chosen to supply. Set
`FIREBASE_WEB_API_KEY_VALUE=<value>` in your shell before running (never
as a `--` argument) to have it set automatically alongside the other two;
otherwise the script prints a one-line manual command for that secret
only at the end, changing nothing else about the run:

```
wrangler secret put FIREBASE_WEB_API_KEY --name arom-production
# paste the value when prompted — Firebase Console -> Project settings ->
# General -> Web API Key, for arom-production-657f2
```

`INVENTORY_SERVICE_EMAIL` is set as a Worker *secret* now (not a plain
variable as in the prior version of this section) — simpler and
consistent with the other two, and the value itself was never sensitive
either way.

**Rotating the password:** generate a new one directly in the Firebase
console (Authentication → find `inventory-service@system.arom.cd` → Reset
password) or via `gcloud`/Admin SDK, then update the
`INVENTORY_SERVICE_PASSWORD` Worker secret with `npx wrangler secret put
INVENTORY_SERVICE_PASSWORD` the same way as initial provisioning. No code
change needed — the route re-authenticates on first use per Worker
isolate.

**Disabling / revoking** (suspected compromise, or the route is being
decommissioned): disable the account immediately in the Firebase console
(Authentication → the account → Disable account) — this takes effect
immediately and does not require redeploying anything, since every
`/api/inventory/qc-release` call re-verifies sign-in. To fully revoke
instead of just disabling, delete the Auth account and its `users/{uid}`
doc; `isInventoryService()` fails closed the moment the custom claim's
account no longer exists or can no longer sign in, so no further access is
possible either way. There is nothing else this account can reach —
its custom claim satisfies no other rule in `firestore.rules`.

## Enabling Google / Facebook sign-in

App code never sees a client ID/secret for either — the whole OAuth
handshake is configured in the Firebase console and Firebase's own
backend handles it. This only needs doing once per Firebase project.

**Google:** Firebase console → `arom-production` → Authentication →
Sign-in method → Google → Enable → set a support email → Save. Nothing
else required — no external app registration.

**Facebook:**
1. Create an app at [developers.facebook.com](https://developers.facebook.com)
   (or use an existing one) and add the **Facebook Login** product to it.
2. Firebase console → Authentication → Sign-in method → Facebook →
   Enable → paste the app's **App ID** and **App Secret** → Save. Firebase
   shows an OAuth redirect URI
   (`https://arom-production.firebaseapp.com/__/auth/handler`).
3. Paste that URI into the Meta app's Facebook Login → Settings →
   **Valid OAuth Redirect URIs**.
4. The Meta app needs to be in **Live** mode (not just Development) for
   accounts other than the app's own testers/admins to be able to sign in.

Local dev against the emulator needs none of this — the Auth emulator
has its own mock IdP flow (an "Auto-generate user information" button in
the popup) that works without any real Google/Facebook account.

## If Firebase Auth needs to be re-enabled on a new project

Enabling Auth (Identity Platform) required linking a GCP billing account
to the project — this wasn't documented anywhere obvious and cost real
time to discover. If this is ever redone from scratch (new environment,
new region, disaster recovery):

1. `gcloud services enable firestore.googleapis.com identitytoolkit.googleapis.com firebasestorage.googleapis.com --project=<id>`
2. `gcloud billing projects link <id> --billing-account=<account-id>` — required before Auth will initialize, even for free-tier usage.
3. Initialize Identity Platform: `POST identitytoolkit.googleapis.com/v2/projects/<id>/identityPlatform:initializeAuth`
4. Enable email/password sign-in: `PATCH identitytoolkit.googleapis.com/admin/v2/projects/<id>/config` with `signIn.email.enabled = true`.

## Frontend deploys

**As of 2026-08-14, `AROM-Production` still isn't deploying
automatically, but the pipeline is built and waiting on credentials.**
Lovable's own pipeline used to deploy it (on every push to the connected
branch); the project has since been disconnected from Lovable, on both
Lovable's dashboard and in the codebase (see
[architecture.md](architecture.md#update-2026-08-14-lovable-removed)).

A `deploy` job now exists in `AROM-Production/.github/workflows/ci.yml`
(sprint 12) — it runs after `verify` passes, only on push to `main`, and
does `bun run build` + `wrangler deploy` via `cloudflare/wrangler-action`.
A root `wrangler.jsonc` pins the Worker name to `arom-production` (Nitro
merges it into the `wrangler.json` it generates at build time, so this
no longer falls back to an auto-generated name).

**Important — this is Cloudflare Workers, not Cloudflare Pages.** Nitro's
`cloudflare-module` preset outputs a Workers entrypoint + `wrangler.json`,
which Pages' generic "build command + output directory" git-integration
flow doesn't understand (would deploy a static shell with no SSR — the
dashboard, storefront auth, and PawaPay server functions would all
break). If setting this up from the Cloudflare dashboard rather than the
GitHub Actions workflow, use "Workers Builds," not "Pages."

The only thing missing before pushes to `main` actually deploy: two
GitHub repo secrets, from whoever owns the Cloudflare account this
should deploy under —

1. `CLOUDFLARE_API_TOKEN` — scoped to "Edit Cloudflare Workers" (the
   Cloudflare dashboard's own token-creation template covers this).
2. `CLOUDFLARE_ACCOUNT_ID` — visible on the right sidebar of any
   Cloudflare dashboard page.

Add both under `AROM-Production` → Settings → Secrets and variables →
Actions. The next push to `main` after that deploys automatically; no
further setup needed.

Until then, a manual deploy from a machine with `wrangler` authenticated
still works: `bun run build && cd .output/server && npx wrangler deploy`
— the deploy has to run from `.output/server` (where Nitro writes the
generated `wrangler.json` with the real `main`/`assets` paths), not repo
root, where only the name-pinning `wrangler.jsonc` lives.
