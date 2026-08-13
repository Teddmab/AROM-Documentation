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

Nothing to do — Lovable deploys `AROM-Production` to Cloudflare Workers on
every push to the connected branch automatically. GitHub Actions CI on
that repo is verification-only (lint/typecheck/build); it does not deploy.
