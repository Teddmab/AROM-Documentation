# AROM Mobile Completion Plan — Progress Tracker

Reconciled against actual repository state on **2026-09-18** — a fresh,
independent closure audit across all four repos (git log/status, GitHub
Actions run history via `gh run list`/`gh run view`, live Cloudflare
secret inventory via `wrangler secret list`, and the live Firestore rules
actually deployed to `arom-production-657f2`, fetched directly via the
Firebase Rules REST API and diffed against every local commit). This file
is git-tracked (currently shows as modified, `M`, not untracked — a
correction to this file's own prior claim below) and should be updated at
the end of every sprint, not just read at the start.

**Quick summary for external review (2026-09-18):** The sprint-completion
picture below (01–10) is **substantially unchanged in spirit** from
2026-09-17, but a same-day audit found something more fundamental
underneath it: **almost none of the last month's committed work in any of
the three code repositories has ever reached a shared or deployed state.**
See "⚠️ Production deployment integrity" immediately below — read this
before trusting any "done"/"landed on main" language elsewhere in this
file. Sprints 01, 02, 04, 05 done (locally). 03, 06 partial. **07 is now
closed**, via a documented decision that supersedes the sprint's original
"temporary redirect" ask (see Sprint 07 below) — plus two further
un-incorporated redesigns (work-queue, finished-stock honesty) landed the
same day. 08 partial: Steps A–D are genuinely built and tested, but nine
distinct things stand between that code and it actually running in
production (enumerated below) — none of which are business decisions,
all of which are mechanical/process gaps. 09 blocked on business
approvals, not started. 10 not started; Android QA build ready, iOS
TestFlight build now finished (upgraded from "in progress").

## ⚠️ Production deployment integrity (found 2026-09-18) — read first

This section exists because every other status in this file was written
assuming "committed" ≈ "on `main`" ≈ "deployed." None of those equivalences
hold right now:

1. **Nothing from the last month is pushed to GitHub, in any of the three
   code repos.** `origin/main` is: AROM-Mobile at `eb9f4a0` (2026-09-01,
   **30 commits** behind local `main`, `d034b66`); AROM-Backend at
   `d0e7350` (2026-08-20, **10 commits** behind local `main`, `1ec143c`);
   AROM-Production at `c44bcc6` (2026-08-16, **17 commits** behind local
   `main`, `a33bda1`). No CI has run on any of this work in its real,
   shared environment.
2. **AROM-Production's Cloudflare Worker deploy has a 100% historical
   failure rate.** Every CI run that ever included a `deploy` job (10/10,
   checked via `gh run view --json jobs`) failed on the same error:
   `wrangler deploy` in `.output/server` finds both that directory's own
   generated `wrangler.json` and a root-level `.wrangler/deploy/
   config.json` (Nitro's `cloudflare-module` preset writes both, on every
   `bun run build`) and refuses to guess which base path governs. The
   live Worker has been running commit `b34ea1ac08` (2026-08-14) — the
   last push whose deploy step actually succeeded — ever since.
   **Root cause reproduced locally** (`wrangler deploy --dry-run` in
   `.output/server` fails identically) **and the smallest fix verified**
   (`wrangler deploy --config wrangler.json` resolves it cleanly, dry-run
   confirmed) — committed to a new, local-only, unpushed branch
   `chore/wrangler-deploy-config-fix` (worktree at
   `/tmp/claude-501/arom-production-wrangler-fix`, commit `beed569`).
3. **AROM-Backend's `deploy-rules.yml` has deployed to the wrong Firebase
   project on every run in its history, including every prior "success."**
   `firebase deploy --project arom-production` has been hardcoded since
   this workflow's first commit (`4df0c76`, 2026-08-13) and was never
   updated — but `arom-production` and `arom-production-657f2` (the
   project `.firebaserc`, AROM-Mobile, and AROM-Production all actually
   point at) are **two different Firebase projects** (confirmed via
   `firebase projects:list`: project numbers `66641088420` vs
   `227952105868`). Correcting `.firebaserc`'s own default on
   2026-08-16 never fixed this, because an explicit `--project` flag
   always overrides it. **Smallest fix identified and committed** to a
   new, local-only, unpushed branch `fix/deploy-rules-wrong-project`
   (worktree at `/tmp/claude-501/arom-backend-project-fix`) — changes the
   flag to `--project arom-production-657f2`; not executed against
   production, per this audit's own scope.
4. **The live rules on the real project (`arom-production-657f2`) were
   last updated 2026-09-16T09:46 UTC — manually, outside any CI —
   and contain none of Sprint 08's Step B/C/D security model.** Fetched
   directly via the Firebase Rules REST API and diffed against every
   local commit: live rules have **zero** occurrences of
   `isInventoryService` or `isValidOrderStatusTransition`; `stockBalance`
   and `stockLotBalance` have **no match block at all** (default-denied);
   `stockPF` create is still the older `isAdmin() || isProductionStaff()`
   direct-client-write predicate, not inventory-service-only; `orders`
   update still allows any admin/staff to set `status`/`reservation` to
   anything, with none of `684702e`'s or `d7385bf`'s narrowing. The
   `harvestOffers` webhook-read fix (`1ec143c`) is also absent live.
5. **The production `inventory-service` Firebase identity does not
   exist.** Checked directly against `arom-production-657f2` via a
   scoped existence-only lookup (no credentials printed or rotated):
   `inventory-service@system.arom.cd` → `auth/user-not-found`. (For
   contrast, `mombongo-webhook@system.arom.cd` **does** exist, is
   active, and last signed in 2026-09-17 — that identity is genuinely
   provisioned; inventory-service is not.)
6. **The live production Worker is missing 3 of the 3 secrets the new
   trusted routes require.** `wrangler secret list --name arom-production`
   returns exactly two secrets: `MOMBONGO_WEBHOOK_EMAIL`,
   `MOMBONGO_WEBHOOK_PASSWORD`. Grepping the code for every `env.*`
   reference shows the trusted routes also need `INVENTORY_SERVICE_EMAIL`,
   `INVENTORY_SERVICE_PASSWORD`, and `FIREBASE_WEB_API_KEY` — none of
   which are configured on the live Worker.
7. **The currently-shipped production mobile build cannot reach any real
   backend for these flows anyway, independent of all of the above.**
   `93b7e0b` (the commit behind the finished iOS build,
   `fbca0b40-2828-4758-9dc5-7ecbd7e0e947`) already has Sprint 08 Steps
   B/C/D's mobile-side trusted-route rewrite as an ancestor (confirmed:
   `git merge-base --is-ancestor` true for all four of `e3ea653`,
   `01cb75e`, `e5f43a3`, `38ef0db`) — the code calls real
   `fetch(`${AROM_PRODUCTION_URL}/api/inventory/...`)` routes, not direct
   Firestore writes. But `eas.json`'s `production` build profile sets no
   `env` block at all (unlike `preview`/`qa-apk`, which both explicitly
   set `EXPO_PUBLIC_AROM_PRODUCTION_URL`), and `eas env:list production`
   confirms **zero** EAS-server-side variables are configured for that
   environment either. The shipped build's `AROM_PRODUCTION_URL` almost
   certainly baked in as the code's own fallback,
   `"http://localhost:8080"` — meaning every QC-release and order
   confirm/cancel/fulfil call in the currently-installed production app
   silently fails against a nonexistent local address. This is
   independent of, and adds to, points 1–6: fixing all of them still
   would not help any already-installed copy of this build; a **new**
   production build with the URL configured would be needed too.

**None of these are business decisions.** They're mechanical/process gaps
— a config typo, a missing CI flag, an unset build-profile variable, an
unprovisioned service account — but together they mean the honest answer
to "is Sprint 08 in production" is **no, not in any part**, regardless of
how much of the code is written, tested, and committed. The full
compatibility matrix, safe cutover order, rollback steps, smoke tests,
and explicit approval points from this audit live in the 2026-09-18
session transcript/report (not yet its own tracked doc — a reasonable
follow-up would be to promote it to a `deployment-readiness.md` alongside
this file); this section is the durable summary of what it found.

## Deployment-safety preparation batch (2026-09-18, later same day)

Prepares the cutover without deploying, merging, provisioning, or
distributing anything. Every item below is committed and pushed to a
**non-`main` branch** confirmed, both by reading the workflow definitions
and empirically (checked `gh run list` immediately after each push — zero
new runs fired anywhere), to trigger no deploy.

**The originally-proposed Worker-then-Rules (or Rules-then-Worker) cutover
was rejected** — either order has a window where the live dashboard and
the live Rules disagree about what an order write looks like. Replaced
with a three-stage transitional cutover (transitional Rules → new
Worker/dashboard → final strict Rules); see `firestore.transitional.rules`
below for the full design and `tests/rules.transitional-orders.test.mjs`
for the proof.

**AROM-Backend** — `firestore.transitional.rules` (new): `firestore.rules`
plus exactly one temporary allowance on `orders/{id}.update`, preserving
the currently-*live* dashboard's unrestricted direct-write shape
side-by-side with the new `isInventoryService()` trusted path. Authorizes
no new actor type — Personnalisé/missing-poste staff already satisfy
`isUnscopedStaff()` under every one of the three rulesets (live,
transitional, final), a pre-existing, unrelated policy this change
doesn't touch. Removal criteria documented on the branch itself. 13 new
emulator tests (`tests/rules.transitional-orders.test.mjs`) prove: the
legacy write succeeds, the new trusted write succeeds, and every actor
already denied under the final policy (scoped-away postes, inactive
accounts, non-owning partners, unauthenticated) stays denied. A
cross-reference comment added to `tests/rules.test.mjs`'s own equivalent
test makes "removed once final" explicit. Also fixed, as a byproduct: two
test files sharing one emulator instance were racing on the Storage
emulator's ruleset load (`vitest.config.mjs`, `fileParallelism: false`) —
unrelated to rules logic, but was making the suite flaky. **301/301
passing** (288 existing + 13 new).

**AROM-Mobile** — `src/lib/config/aromProductionUrl.ts` (new): centralizes
the 5 previously-duplicated inline
`env var ?? "http://localhost:8080"` declarations into one
`getAromProductionUrl()`, called lazily at the point of use, which throws
instead of silently falling back whenever a release build (`!__DEV__`)
would resolve to nothing, to a localhost/emulator host, or to the QA
Worker while not itself the QA target. `eas.json`'s `production` profile
now sets the real Worker URL (matching `preview`'s own value) — the
concrete fix for the "shipped build can't reach any backend" finding
above. New/extended tests: `__tests__/aromProductionUrl.test.ts` (the
resolver's own logic, every dev/release × configured/missing/local/QA
combination) and `__tests__/easConfig.test.ts` (static checks that
`production`/`preview`/`qa-apk` all resolve to a real, non-local URL).
**1770/1770 passing**, `tsc`/`eslint`/`expo export --platform web` all
clean.

### Revised three-stage cutover order

1. **Transitional Rules** — deploy `firestore.transitional.rules` (as
   `firestore.rules`) to `arom-production-657f2`. Both the live dashboard
   and the not-yet-live new Worker/dashboard can run against it
   simultaneously. *Rollback*: redeploy the current live ruleset (Console
   "restore," or redeploy ruleset `3e942b16`, 2026-08-31 — the release
   immediately before today's live one) — a config-only revert, no code
   involved.
2. **New Worker/dashboard + trusted-route smoke tests** — merge the fixed
   `chore/wrangler-deploy-config-fix` + `stabilize/main-sync` into
   AROM-Production `main`. Smoke test: `wrangler deployments list` shows
   the new deployment active; one validation-only call to a trusted route
   (expect a clean validation error, not 500/permission-denied); one real
   order confirm through the dashboard succeeds end-to-end. *Rollback*:
   `wrangler rollback` to the prior deployment (`b34ea1ac08`) — near-instant,
   Cloudflare retains deployment history.
3. **Final strict Rules** — deploy the real `firestore.rules` (no
   transitional branch) once step 2 is confirmed live and serving.
   Removes the legacy escape hatch. Smoke test: diff live rules against
   local `firestore.rules`, expect zero diff; re-run
   `tests/rules.test.mjs`'s "an ordinary ADMIN can no longer flip status
   directly" against the live project's behavior (or trust the emulator
   proof, since the ruleset is byte-identical). *Rollback*: redeploy
   `firestore.transitional.rules` again — a config-only revert, buys time
   without losing the new Worker/dashboard.

Steps 1 and 3 depend on the wrong-project fix (`fix/deploy-rules-wrong-project`)
being merged first, or they redeploy to the decoy `arom-production` project
again.

Separately, before any of steps 1–3: **provision `inventory-service` for
`arom-production-657f2`** and **set the Worker's 3 missing secrets**
(`INVENTORY_SERVICE_EMAIL`, `INVENTORY_SERVICE_PASSWORD`,
`FIREBASE_WEB_API_KEY`) — not a deploy, but must happen before step 2's
trusted routes can do anything useful once live. And independently,
**cut a new mobile production build** once `eas.json`'s fix (above) is
confirmed — the currently-installed build cannot reach any backend for
these flows regardless of steps 1–3.

### Remote branches pushed (all non-`main`, zero deploys triggered — confirmed via `gh run list` immediately after each push)

| Repo | Branch | Commit | PR | CI result |
|---|---|---|---|---|
| AROM-Mobile | `stabilize/main-sync` | `3cc48d2` | [#2](https://github.com/Teddmab/AROM-Mobile/pull/2) | `verify` ✅ pass (lint/typecheck/test) |
| AROM-Backend | `stabilize/main-sync` | `1ec143c` | [#5](https://github.com/Teddmab/AROM-Backend/pull/5) | no checks configured for PRs on this repo (`deploy-rules.yml` is push-only) |
| AROM-Backend | `fix/deploy-rules-wrong-project` | `a6e096b` | [#6](https://github.com/Teddmab/AROM-Backend/pull/6) | no checks configured (same reason) |
| AROM-Backend | `feat/transitional-order-rules` | `9aada50` | [#7](https://github.com/Teddmab/AROM-Backend/pull/7) | no checks configured (same reason); 301/301 local |
| AROM-Production | `stabilize/main-sync` | `a33bda1` | [#21](https://github.com/Teddmab/AROM-Production/pull/21) | `verify` ✅ pass; `deploy` correctly **skipped** (not a push event) |
| AROM-Production | `chore/wrangler-deploy-config-fix` | `beed569` | [#22](https://github.com/Teddmab/AROM-Production/pull/22) | `verify` ✅ pass; `deploy` correctly **skipped** |

**Update, same day: PRs #5, #6, #7 above were merged into `AROM-Backend`
`main` by the account owner shortly after this table was written** — a
decision within their own authority, not something this session did.
`AROM-Production`'s equivalent PRs (#21, #22) were **not** merged and
remain open; `AROM-Mobile`'s #2 was merged but carries zero deploy risk
either way (no deploy job exists in that repo's CI).

## ⚠️ Incident: merging #5+#6 auto-fired the (still-unsafe) rules deploy twice — both failed, live rules unchanged

Merging `stabilize/main-sync` (#5, brings the final strict `firestore.rules`
onto `main`) and `fix/deploy-rules-wrong-project` (#6, corrects the
`--project` flag) landed the OLD `deploy-rules.yml` — still push-triggered
at that point, since its replacement (below) hadn't merged yet — squarely
on its own trigger condition. It fired twice, once per merge commit
(2026-09-18 10:36 and 10:37 UTC):

- Run 1 (before #6's fix applied): `firebase deploy --project arom-production`
  — failed: `Deploy target prod-default not configured for project
  arom-production` (a storage-target config gap on the decoy project;
  never reached Firestore rules at all).
- Run 2 (after #6's fix applied): `firebase deploy --project
  arom-production-657f2` — the **real** project this time — failed:
  `HTTP 403: Caller does not have required permission to use project
  arom-production-657f2. Grant the caller the roles/
  serviceusage.serviceUsageConsumer role...`. **The GitHub Actions
  service account (`FIREBASE_SERVICE_ACCOUNT_KEY`) has no IAM permission
  on the real production project at all** — a previously-unknown, newly
  discovered blocker in its own right, and (by lucky accident, not by
  design) the only reason this run didn't actually deploy the final
  strict rules straight past the transitional migration.

**Confirmed directly, twice, via the Firebase Rules REST API (not
assumed): the live ruleset is still `b4dbfaf2`, updated
2026-09-16T09:46:39Z — byte-identical to before either run.** Nothing
was deployed. This was the exact risk the "make Rules deployment
selection safe" work below was already underway to close — see that
section for the fix, now itself merge-ready. **`main` is still armed
with the old unsafe workflow as of this writing** — the next push
touching `firestore.rules` will try again, and will succeed the moment
anyone fixes the IAM gap above without knowing about this risk. Merging
PR #9 below is now urgent, not merely tidy.

## A. Root cause of the Personnalisé contradiction

A prior version of this tracker's report asserted, in a summary table,
that Personnalisé succeeds under **final** Rules for a direct order
confirm — presented as a possible security defect. **This was never
backed by a test.** `grep -n "Personnalisé" tests/rules.test.mjs` shows
every existing Personnalisé case in that file targets other collections
(producteurs, stockPF-adjacent, tasks) — the `orders` describe block had
zero.

**Root cause**: conflating actor *recognition* with operation
*authorization*. `isUnscopedStaff()` (`firestore.rules:72-76`) already
returns `true` for Personnalisé, a missing poste, or an unrecognized
poste string — a real, pre-existing, unrelated policy, documented on
that function itself, present identically in the live/transitional/final
rulesets. But `orders/{id}.update`'s further AND-conditions (reservation
must stay unchanged; status must stay unchanged or go pending→cancelled)
apply identically to *every* actor the top-level
`isAdmin() || isUnscopedStaff() || isCommercialStaff()` OR recognizes —
being unscoped-staff only gets an actor into that OR, it grants no
additional operation-level privilege beyond what Admin/Commercial
themselves have. There is exactly one rule block that can authorize an
`orders/{id}` write in either ruleset (confirmed: `grep -n "match
/orders"` — no second or wildcard match block exists anywhere in this
schema).

**Corrected, proven claim**: Personnalisé, missing-poste, and
unknown-poste behave **identically** to Admin and Commercial under both
the final and transitional policies — none of them is a defect, because
none of them differs from the other two.

### Actor × operation matrix — proven directly (72 isolated emulator assertions, `tests/rules.orders-actor-matrix.test.mjs`)

| Actor | pending→confirmed | confirmed→fulfilled | confirmed→cancelled | pending→cancelled | reservation mutation | unrelated-field mutation |
|---|---|---|---|---|---|---|
| Admin | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ✅ / Trans: ✅ | Final: ❌ / Trans: ❌ | Final: ✅ / Trans: ✅ |
| Commercial | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ✅ / Trans: ✅ | Final: ❌ / Trans: ❌ | Final: ✅ / Trans: ✅ |
| Personnalisé | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ✅ / Trans: ✅ | Final: ❌ / Trans: ❌ | Final: ✅ / Trans: ✅ |
| Missing poste | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ✅ / Trans: ✅ | Final: ❌ / Trans: ❌ | Final: ✅ / Trans: ✅ |
| Unknown poste | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ❌ / Trans: ✅ | Final: ✅ / Trans: ✅ | Final: ❌ / Trans: ❌ | Final: ✅ / Trans: ✅ |
| Inactive admin | Final: ❌ / Trans: ❌ | Final: ❌ / Trans: ❌ | Final: ❌ / Trans: ❌ | Final: ❌ / Trans: ❌ | Final: ❌ / Trans: ❌ | Final: ❌ / Trans: ❌ |

Every ✅/❌ above is one real `assertSucceeds`/`assertFails` call against
a live emulator, not an inference. The exact predicate authorizing every
✅ cell: final policy's own restricted branch (pending→cancelled,
unrelated-field) or — transitionally only — the three narrow legacy
branches described in section C below.

## B. Rules deployment selection — redesigned

`deploy-rules.yml` (push-to-main-triggered, wrong-project-targeted since
its first commit) is **replaced by two workflows**, on branch
`feat/safe-rules-deploy-workflow`:

- **`validate-rules.yml`** — push/PR, any branch, no deploy step exists
  in the file at all. Runs the full emulator suite (394 tests), which
  covers both `firestore.rules` and `firestore.transitional.rules`.
- **`deploy-rules-production.yml`** — `workflow_dispatch` only (no
  `push`/`pull_request` trigger anywhere in it — confirmed both
  statically and by the fact that merging #5+#6 could only fire the OLD
  workflow, never this one). Gated on the `production-rules` GitHub
  Environment (reviewer approval — configure once in Settings →
  Environments; not expressible in the YAML itself). Requires a `policy`
  choice input (`transitional`|`final`, no default) plus a hand-typed
  `confirm_policy` that must match; fails closed on anything else.
  Copies the selected file into an isolated scratch directory with its
  own `firebase.json`/`.firebaserc` — never overwrites the repo's own
  tracked `firestore.rules`. Target project is the literal
  `arom-production-657f2` in the deploy command itself, never an input.
  Logs the selected policy + source commit, then runs the new
  `scripts/verify-deployed-rules.mjs` to fetch the live ruleset via the
  Rules REST API and diff it byte-for-byte against the selected file,
  failing the job on any mismatch.

19 new static tests (`tests/deployWorkflow.test.mjs`) prove every one of
the 5 requested properties. **CI-verified for real**: PR #9's `validate`
job passed on an actual GitHub-hosted runner (2 real bugs caught and
fixed along the way — `firebase-tools` wasn't installed, then the
emulator needed JDK 21 which `ubuntu-latest`'s default doesn't have;
neither had ever been needed before since no prior workflow ran the
emulator in CI at all).

## C. Transitional scope — verified against the exact live ruleset, then narrowed

Fetched the real deployed ruleset via the Firebase Rules REST API
(`arom-production-657f2`, release `b4dbfaf2`, 2026-09-16) — not a local
historical file. **The live `orders/{id}.update` is exactly
`isAdmin() || isUnscopedStaff() || isCommercialStaff()`, with zero field
restriction** — functionally identical to this branch's first draft of
the transitional escape hatch. So that first draft added no *new* risk
beyond the status quo — but "no new risk" isn't "as narrow as possible."

Read the actual currently-deployed dashboard source
(`git show b34ea1ac08:src/routes/dashboard.tsx` in AROM-Production) to
find its exhaustive real write shapes: exactly three —
`{status: pending→confirmed, deliveryDate?}`,
`{status: confirmed→cancelled}` (status only), and
`{status: confirmed→fulfilled, fulfilledAt}`. A plain pending→cancelled
write is already covered by the final policy's own carve-out and needs
no transitional help. **Narrowed the escape hatch to precisely those
three shapes** (`request.resource.data.diff(...).affectedKeys()`, the
same idiom already used 5 other places in `firestore.rules`) — arbitrary
document-field mutation, and specifically an unrestricted reservation
write, is **not** required for compatibility and is **no longer
granted**, even transitionally (proven: the "reservation mutation only"
column above is ❌ under both final and transitional).

The one shape that genuinely bypasses reservation release
(confirmed→cancelled, status only) is kept — removing it would break the
live "Annuler" button on a confirmed order for the entire cutover
window — but it's now the narrowest form possible (status field only,
nothing else). If, once this is deployed, an audit of writes made during
the window is wanted: query `orders` where `status == 'cancelled'` and
`updatedAt`/write-time falls inside the transitional deploy window, and
confirm each such document either has no `reservation` field or one
matching a real, since-reconciled release — this only matters for
documents cancelled from `confirmed` (a pending cancel never had a
reservation to release).

## Revised cutover order (supersedes the two-blocker version above)

0. **Merge PR #9 (`feat/safe-rules-deploy-workflow`) — now urgent**, since
   `main` is still armed with the old unsafe workflow (see incident
   above).
1. Merge PR #8 (`feat/transitional-order-rules`'s narrowing fix) —
   updates the (already-merged, over-broad) transitional file to the
   narrow one before it's ever selected for deploy.
2. Fix the IAM permission gap found above (grant
   `roles/serviceusage.serviceUsageConsumer`, or equivalent, to the
   `FIREBASE_SERVICE_ACCOUNT_KEY` service account on
   `arom-production-657f2`) — nothing can deploy without this regardless
   of workflow safety.
3. Run `deploy-rules-production.yml` with `policy=transitional`.
4. Merge AROM-Production's `stabilize/main-sync` + `chore/wrangler-deploy-config-fix`
   (PRs #21/#22) → live Worker deploy. Smoke test as before.
5. Run `deploy-rules-production.yml` with `policy=final`.
6. Configure the mobile production URL + cut a new build (independent,
   any time before real users need the trusted routes to work).

## Remaining explicit approval points
- Merging PR #9 (urgent — closes the live risk).
- Merging PR #8.
- Granting the IAM role for the deploy service account.
- Each `workflow_dispatch` run of `deploy-rules-production.yml` (gated by
  the `production-rules` environment's required reviewers, once
  configured).
- Merging AROM-Production's #21/#22 → live Worker deploy.
- Provisioning `inventory-service` + the 3 missing Worker secrets.
- Cutting/distributing a new mobile production build.

## Status legend

- ✅ **Complete and committed** — verified against real code/tests/rules; committed (locally at minimum — see the push-status column).
- 🟡 **Partially complete** — real, shipped work exists; a specific listed gap remains.
- 🔁 **Superseded** — the original instruction was replaced by a later, documented decision.
- ⬜ **Not started**.
- 🚧 **Blocked (business)** — a real business/product decision is required before more can be built.
- 🛠 **Implemented but not deployed** — code is written, committed, and tested; the deploy pipeline, a config value, or a provisioning step stands between it and production. Never a business gate.
- 🔍 **Remaining verification only** — implementation appears done; needs a native/device pass.

## Summary table

Pushed-to-origin column added 2026-09-18 — see "⚠️ Production deployment
integrity" above for why this is now tracked as its own dimension,
separate from "done."

| # | Prompt | Status | Pushed to origin? | Evidence |
|---|---|---|---|---|
| 01 | `01-documentation-and-flags.md` | ✅ Complete | ❌ No (Mobile 30 commits ahead) | AROM-Mobile `login.tsx`, `oauthConfig.ts`, `getPaymentGateway.ts`, `easConfig.test.ts`; AROM-Documentation `architecture.md`/`data-model.md`/`roadmap.md` content (dirty, uncommitted, but correct) |
| 02 | `02-freshness.md` | ✅ Complete | ❌ No | `refreshCoordinator.ts`, `useAutoRefresh.ts`, `SyncStatus.tsx` + 4 test files |
| 03 | `03-security.md` | 🟡 Partial | ❌ No | poste deny-by-default done; **missing**: read-only staff-poste audit report/script, and `isValidStockMP`/`isValidClients`/`isValidVentes`/`isValidMarketing`/`isValidCharges` in `AROM-Backend/firestore.rules` — both confirmed still absent, re-verified 2026-09-18 |
| 04 | `04-navigation-and-drafts.md` | ✅ Complete | ❌ No | `activeMode.tsx`, "Passer en mode ..." wording, reception→producer draft-return, payment-waiting safe-to-leave screen, `activeMode.test.tsx` |
| 05 | `05-reception-and-production.md` | ✅ Complete | ❌ No | Reception: `ProducerStep.tsx`, `receptionStatus.ts`. Quality journey: `productions.tsx`, `qualityRepository.ts` head-control walk, `resolve.tsx`. Production/packaging redesign: `25996b9`, `4e1c4c9`. Lot-detail redesign: `eae9cf4`/`bdfcfad`/`fe4ec14`/`1f53ccb`. No reception-redesign kit exists anywhere beyond this — confirmed 2026-09-18, nothing outstanding here |
| 06 | `06-sales.md` | 🟡 Partial | ❌ No | Wizard order close but client is still mandatory, lot picker is manual/mandatory and shows lot IDs, no +/- controls, **no availability check of any kind exists** — re-verified byte-for-byte unchanged 2026-09-18, even after `d034b66` added a real `stockBalance` sync category for PRODUCTION/SALES (that category is not consumed anywhere in the sale wizard) |
| 07 | `07-admin.md` | 🔁 **Superseded, closed** | ❌ No | `admin-operations.tsx` (158 lines) deliberately kept as a real, data-free navigation directory instead of a "temporary redirect," per its own doc comment and `design-references/admin/overview-redesign/AROM_ADMIN_OVERVIEW_REFERENCE.md` — zero remaining nav entry points, closed by `c118ff2`/`b637b40`/`86aef83` (2026-09-17, before this tracker's prior revision was even written). Two further un-incorporated redesigns landed the same day, also not yet folded into this pack: the work-queue replacement of "Consignes" (`cbe18b3`, spec at `design-references/work-queue/`) and the finished-stock dedup/honesty fix (`d034b66`, spec at `design-references/stock-pf/`) |
| 08 | `08-finished-product-stock.md` | 🟡 Partial (Steps A–D 🛠 implemented, not deployed — see above) | ❌ No | **Steps A–D** (schema, QC-release receipts, order reservation/cancel/fulfil, dashboard integration) genuinely built, tested, and — per the "⚠️ Production deployment integrity" section above — confirmed **not live in production** for 7 independent reasons (unpushed, broken Worker deploy, wrong rules-deploy target, live rules missing the whole security model, no inventory-service identity, missing Worker secrets, no backend URL in the shipped mobile build). The old "dashboard fix not merged" framing is stale: `93ccda3`'s content is confirmed byte-identical on local `main` via `a33bda1` (cherry-pick) — the real remaining gap is deployment, not merge. **Steps E/F** (direct-sale, UI) **not started** — confirmed zero trace of any `direct-sale` endpoint or `CommercialisationSection` repoint |
| 09 | `09-partner-orders.md` | 🚧 Blocked (business) | n/a | Explicitly gated on business approvals (payment/refund ownership, formats/prices, reservation duration/release policy, customer data, partner naming) — no approval record found anywhere in AROM-Documentation, re-confirmed 2026-09-18 |
| 10 | `10-field-validation.md` | ⬜ Not started | n/a | Depends on 06/07/08/09 being substantially done first. Android QA APK ready. iOS build `fbca0b40` now **FINISHED** (was "in progress" as of 2026-09-17) — TestFlight submission/testers still unconfirmed |

## Detail per sprint

### 01 — Documentation and flags — ✅ Complete
- `AROM-Documentation/architecture.md:127,138` documents AROM Mobile in the topology.
- `data-model.md:209-230` documents `producerInvoices`/`harvestOffers`/`harvestInvoices`, and records the decision that a future `partnerOrders` domain must never reuse `producerInvoices`/`mombongoInvoices`.
- `roadmap.md:280` marks the old combined-sprint Mombongo vision superseded, with corrected status.
- `AROM-Mobile/src/app/login.tsx:53-57` + `src/lib/firebase/oauthConfig.ts` hide Google/Facebook buttons when client IDs are unconfigured.
- `src/features/payments/getPaymentGateway.ts` + `eas.json` (`preview` only, never `production`) + `__tests__/easConfig.test.ts` guard the Mombongo simulation flag.
- Note: the six core AROM-Documentation files show as `git status` modified/untracked — pre-existing, not part of any commit yet. Content is correct regardless.

### 02 — Freshness — ✅ Complete
- `src/features/sync/refreshCoordinator.ts` — `refreshRolePlan()`, in-flight dedup map, `FOCUS_REFRESH_THROTTLE_MS = 60_000`, `success/partial/failed/offline/throttled/deduped` outcomes.
- `src/features/sync/useAutoRefresh.ts` — foreground (`AppState` "active") + focus-triggered (ADMIN) refresh, obsolete-result guarding via a key ref.
- `src/features/home/components/SyncStatus.tsx` — staleness threshold + last-sync display.
- Tests: `refreshCoordinator.test.ts`, `useAutoRefresh.test.tsx`, `SyncStatus.test.tsx`, `refreshFreshness.test.ts`.

### 03 — Security — 🟡 Partial
Done:
- `src/features/auth/roleAuthorization.ts`'s `getAuthorizedRoles()` denies by default for missing/unrecognized/"Personnalisé" poste.
- Webhook-only sensitive transitions preserved (`isValidMombongoWebhookTransition`, `isValidInvoiceTransition`, `isValidHarvestInvoiceTransition`).
- Strong emulator Rules test coverage for existing validated collections (`AROM-Backend/tests/rules.test.mjs`).

Not done:
- No read-only report (script or screen) listing active accounts with a missing/unsupported/"Personnalisé" poste.
- No migration-impact report or approval gate for existing unscoped accounts.
- `AROM-Backend/firestore.rules` has no `isValidStockMP`/`isValidClients`/`isValidVentes`/`isValidMarketing`/`isValidCharges` — these 5 collections are role-gated only, no field-shape validation (unlike `isValidProducteur`/`isValidProduction`/etc., which already exist).

This remainder is small, self-contained, and does not conflict with sprint 08 — a reasonable follow-up sprint.

### 04 — Navigation and drafts — ✅ Complete
- `src/features/home/activeMode.tsx` (`ActiveModeProvider`) is the sole owner of `activeMode`, re-validated against `getAuthorizedRoles` on every switch.
- `"Passer en mode ..."` wording confirmed in `more.tsx:107` and `AdminReadOnlyGate.tsx`.
- Reception→producer draft-return: `reception-new.tsx` passes `returnTo`/`receptionDraftId`; `producteur-new.tsx` reads it, runs a "minimal mode," and returns via `onUseForReception`. Tested in `ProducteurWizardScreen.test.tsx`.
- Payment-waiting safe-to-leave: `producer-invoice/[id]/pay.tsx`.
- Per-wizard exit-protection tests exist across `ReceptionWizardScreen.test.tsx`, `ProductionWizardScreen.test.tsx`, `SaleWizardScreen.test.tsx`, plus `activeMode.test.tsx` for mode-switch/unauthorized-refusal.

### 05 — Reception and production journeys — ✅ Complete
**Reception**: `ProducerStep.tsx` ("Qui livre les fruits ?", recent producers, local search, add-new), minimal producer creation (`IdentityStep`/`ContactStep`/`ActivityStep`, skips photo mid-reception), auto-return to the same draft, `VerifyStep.tsx` plain-language recap, `receptionStatus.ts`'s explicit Brouillon/En attente/Synchronisation/Synchronisé/Échec state machine.

**Quality/production journey**: `productions.tsx` derives À terminer / Contrôle qualité à faire / Productions récentes sections from sync history + quality lots, one action per row. `StatusBadge.tsx` labels match spec exactly. `qualityRepository.ts` walks the `resolvesId` chain to find the latest *effective* control (not latest-by-timestamp). `quality-control/[id]/resolve.tsx` creates a linked resolution control rather than mutating the original.

**Production quantity/output/packaging redesign** (a deeper elaboration of this same sprint, per `manual-entry-automation-audit.md`): done.
- `4e1c4c9` — visual quantity feedback (fruit-crate/juice-container/sample-container/bottle-count `QuantityVisual`, reception comparison).
- `25996b9` — Quantité utilisée / Résultat obtenu / Embouteillage step redesigns, shared `MeasurementBar`/`QuantityStepperInput`, format-change confirmation.

**Production lot-detail screen redesign** (`production/[id].tsx`, same sprint's "quality/production journey" ground): done, several commits, 2026-09-16/17.
- `eae9cf4`/`bdfcfad`/`fe4ec149` — full redesign: state-driven primary action card, `qualityJourney.ts` (pure, tested, walks quarantine→resolution as one understandable timeline), `LotSummaryFlow.tsx` (fruit/jus/bouteilles summary using three real, size-optimized PNGs via `expo-image`, not the animated `QuantityVisual` used elsewhere), Rendement card sourced only from the immutable `parametresSnapshot`, tappable Origine des fruits, compact Détails du lot grid. Removed "Valeur estimée" (no historically-accurate price snapshot existed to justify it).
- `1f53ccb` — two direct visual-feedback fixes: `LotSummaryFlow`'s three images now align on one row at a consistent height (previously uneven when the bottle column's extra "format" line pushed it taller than the other two) with titles centered underneath; `AppHeader` (the one shared header used on every screen) now shades from a slightly darker tone at the top to the normal background at the bottom.
- Do not reopen without a reproduced regression (per this session's own instruction).

### 06 — Product-first sales — 🟡 Partial (real gap, not superseded)
- `saleDraft.ts`'s `WIZARD_STEPS = ["details", "lots", "client", "payment", "verify"]` — order is close, but:
  - `validateClientStep` still requires `idClient` — client is **mandatory**, not optional as specified.
  - `LotsStep.tsx` is a manual, **mandatory** multiselect showing `production.lot` (the lot ID) directly during normal entry — the opposite of "do not show production-lot IDs during normal sale entry" and "allocate eligible production lots automatically... ask the user only if automatic allocation cannot succeed."
  - `DetailsStep.tsx` uses plain numeric `TextField`s — no large minus/plus controls anywhere in `src/features/sale/components/*`.
  - **No finished-product availability check exists anywhere in the sale flow** — not the old estimate, not the new `stockBalance` model. Nothing to swap; this needs to be built.
- Only 2 commits ever touched this area (`eb9f4a0` MOB-03–11, `99d1129` post-MOB-11 pass) — the sales-specific redesign was never actually done in either.
- **Real dependency confirmed**: the sprint's own text says not to estimate availability and to record the dependency on sprint 08 — automatic FIFO allocation belongs naturally on top of `stockBalance`/`stockLotBalance` (sprint 08 Step E, "direct sales"), so finishing this properly is best sequenced *after* sprint 08's Steps C–E, not before.

### 07 — Administrator experience — 🔁 Superseded, closed (corrected 2026-09-18)
- Four distinct, correctly-wired screens: Home (`AdminOverviewScreen.tsx`), Activity (`/activity`), Operational Summary (`admin-summary.tsx`), Priorities (`admin-priorities.tsx` + detail).
- Home's card already reads "Voir l'activité récente" → `/activity`; zero remaining navigation entry points to `/admin-operations` (confirmed by grep).
- Honesty rules verified: `adminOperations.ts` never infers health from a zero count; `admin-summary.tsx` shows a stale-data banner; `adminSummary.ts` tallies the already-deduped head-control result (no double-counting quarantine + resolution).
- `adminPriorities.ts` uses stable composite keys (e.g. `` `quality-awaiting-${productionId}` ``).
- **Prior "gap" is now closed by a documented design decision, not by building the literal fallback.** `src/app/admin-operations.tsx` is 158 lines (re-measured 2026-09-18) and deliberately kept as a real, data-free navigation directory (module list only, `useHomeGuard()` for auth, zero repository calls) rather than converted to a "temporary redirect" — its own doc comment cites `design-references/admin/overview-redesign/AROM_ADMIN_OVERVIEW_REFERENCE.md` as the source of that decision. Closed by `AROM-Mobile` `c118ff2`/`b637b40`/`86aef83` (2026-09-17) — **before** this tracker's 2026-09-17 revision was even written, so that revision was already stale on arrival.
- **Two further redesigns landed the same day, not originally part of this sprint and not yet folded into this pack:**
  - `cbe18b3` — replaces "Consignes" with a role-aware active work queue (ADMIN "À traiter" / operational "À faire"), per `design-references/work-queue/`. New files: `src/features/home/workQueue.ts`, `src/features/admin/adminWorkQueueExtras.ts`, `src/app/tasks.tsx` rewrite.
  - `d034b66` — fixes a real duplicate-work-item bug in `getLotsToControl` (stale draft-merge guard), gives team-invite queue cards their own illustration instead of reusing a stock asset, and redesigns `/stock-pf` for lifecycle honesty (real onHand/reserved/available, never a fabricated zero) — per `design-references/stock-pf/`. Also fixed live: PRODUCTION/SALES were never given the `stockBalance` sync category despite already being Firestore-authorized to read it.
- **Recommended tracker action**: fold `design-references/admin/overview-redesign/`, `design-references/work-queue/`, and `design-references/stock-pf/` into this pack's own prompt numbering (e.g. as `07a`/`07b`/`08a` amendments) rather than leaving them as orphaned design-reference folders this file never mentions.

### 08 — Finished-product stock and reservations — 🟡 Partial
**Step A (schema) — done.** `AROM-Mobile` `4f757ad` (`StockBalance`/`StockLotBalance` discriminated union, `allocateFifo`, canonical-format mapping), mirrored in `AROM-Backend` `25a2770` (Rules shapes + migration/reconciliation dry-run script) and `AROM-Production` `58b6994` (canonical formats) + `b004818` (QC-gated finished-stock projection fix).

**Step B (QC-release receipts) — done.** `AROM-Mobile` `e3ea653`/`01cb75e`/`e5f43a3` (stockPF write → routed through the trusted inventory service, actor-identity/lost-ack hardening), `AROM-Backend` `fc096fb`/`ea218bc` (Rules restrict `stockPF`/`stockBalance`/`stockLotBalance` writes to the `isInventoryService()` claim only, provisioning script, 239-test Rules suite), `AROM-Production` `7cf6379` (`/api/inventory/qc-release` trusted route + `qcReleaseReceipt.ts` transaction, 556-line test file).

**Steps C/D (order confirm/cancel/fulfil) — done. Dashboard integration gap now closed.**
- `AROM-Production` `17d2277`: `src/lib/inventory/stockBalance.ts` (ported FIFO model), `src/lib/inventory/orderReservation.ts` (`confirmOrderReservation`/`cancelOrderReservation`/`fulfilOrderReservation` — transactional, idempotent, validates every item against a live `products/{id}` doc, allocates FIFO across `stockLotBalance`, writes the `stockPF`/`ventes` bridge rows on fulfil), `src/lib/auth/verifyCommercialInventoryCaller.ts` (admin or "Chargée de Commercialisation"), 3 new trusted routes `/api/inventory/{confirm-order,cancel-order,fulfil-order}.ts`. 93/93 vitest passing, `tsc --noEmit` clean, `vite build` clean (route tree registers all 3).
- `AROM-Backend` `d7385bf`: `firestore.rules` tightens `orders/{id}.update` to a 3-branch rule — `isInventoryService()` + a validated `isValidOrderStatusTransition` for the confirm/cancel/fulfil status+reservation change; ordinary staff keep free edit of every other field; partner pending-order self-cancel unchanged. 10 new Rules tests, suite 283/283.
- `AROM-Backend` `684702e` (**follow-up fix**): `d7385bf`'s ordinary-staff branch required `status` fully unchanged, which unintentionally also blocked staff from cancelling a still-*pending* order (a transition that never touches stock/reservation, and was already a documented-safe carve-out for partners). Extended the staff branch to also allow pending→cancelled, mirroring the partner carve-out. 4 new tests, suite 287/287.
- `AROM-Mobile` `38ef0db`: `orderReservationClient.ts` (new, mirrors `inventoryReleaseClient.ts`), `orderActions.ts` rewritten (`confirmOrder`/`cancelOrder`/`fulfillOrder` call the trusted routes; `cancelPendingOrder` keeps the original direct write for the still-safe pending-order case), `orders.tsx` repointed (confirmed via `OrderDetailModal`'s own JSX that the cancel button is structurally unreachable for non-pending orders, so it calls `cancelPendingOrder` only). Full Jest suite 1525/1525, `tsc`/`eslint` clean, `expo export --platform web` clean.
- `AROM-Production` `93ccda3` (**dashboard integration gap, closed**): `dashboard.tsx`'s "Commandes boutique partenaires" card, the duplicate task-driven `fulfillOrder` helper, and the order-confirm task-completion handler (5 call sites total) repointed to `confirmOrderTrusted`/`cancelOrderTrusted`/`fulfilOrderTrusted`. The still-pending-order cancel button is unchanged (direct write, now confirmed Rules-safe by `684702e` above). Built on a **clean git worktree off `17d2277`** rather than the dirty working tree, since `dashboard.tsx`/`routeTree.gen.ts` there carry large, unrelated, pre-existing uncommitted changes (task-stage-scoping security hardening, reception photo-evidence display, and an uncommitted Mombongo-integration slice) that predate this fix. Added a focused static-protection test (`dashboard.protected-orders.test.ts` — no React component-test infra exists in this repo) asserting no direct write of `status: "confirmed"`/`"fulfilled"` or a `reservation` field remains. 98/98 vitest, `tsc`/`eslint`/`vite build` all clean.
  - **Update (2026-09-17, later same day): landed on `main`.** A dedicated stabilization session first versioned the previously-uncommitted Mombongo-integration slice as `AROM-Production` `2db292a` (`feat(mombongo): version existing invoice integration` — outbound calls, inbound webhook, 5 mobile-facing routes, 152 tests; see `AROM-Documentation/mombongo-integration-audit.md` and `architecture.md`'s "Mombongo integration — status" section), then cherry-picked `93ccda3` on top as `a33bda1`. The other unrelated uncommitted work (`tasks.ts`, `auth.tsx`, `login.tsx`, `join.tsx`, `storefront/signup.tsx`, and dashboard's task-stage-scoping/photo-evidence changes) was preserved, not lost — see branch `recovery/pre-dashboard-repair-dirty-work` on `AROM-Production`. **Sprint 08 Steps C/D, including the dashboard integration gap, are now fully done on local `main`.**
  - **Correction (2026-09-18): "done and on `main`" was true only of the local repository.** `a33bda1` (and everything under it, including `2db292a`, `d7385bf`, `684702e`) has never been pushed to `origin/main` — confirmed via `git rev-list --left-right --count origin/main...main` (17 commits ahead). Even a push alone would not make this live: AROM-Production's Cloudflare Worker deploy has a 100% historical CI failure rate (wrangler config conflict, root-caused and fixed in an unpushed branch — see "⚠️ Production deployment integrity" above), and the corresponding `firestore.rules` changes (`d7385bf`/`684702e`, plus Step A/B's `25a2770`/`fc096fb`/`ea218bc`) were never deployed to the real project either (AROM-Backend's `deploy-rules.yml` has always targeted the wrong Firebase project — also root-caused and fixed in an unpushed branch). The live production Firestore rules, fetched directly 2026-09-18, contain none of Steps B/C/D's security model, and the production `inventory-service` Firebase identity does not exist. **Steps C/D are implemented and tested, not deployed.**

**Steps E/F — not built.** Fully specified, pre-approved architecture exists at `AROM-Documentation/automation-engine.md` lines 217-476 ("Reservation and balance projections — approved architecture (Sprint 08, 2026-09; not yet implemented)") and its own stated build order at line 664: *"Step A ... before B ... before C/D (order reservation/fulfilment, meaningless without B already trusted) before E (direct sales) before F (UI)."*
- **E — direct sales**: repoint `AROM-Production`'s `CommercialisationSection` and `AROM-Mobile`'s `sale-new` at a `direct-sale` trusted endpoint.
- **F — UI**: sprint 06's own remaining sales-wizard work, naturally sequenced after E.

→ **Next up: Step E (direct sales)** — mobile Mombongo invoice reflection (below) is done.

## Mombongo invoice reflection — ✅ Done (2026-09-17, later same day)

Backend baseline (`AROM-Production` `2db292a` + `a33bda1`, above) plus the mobile side that was missing: `AROM-Mobile` `a1cab49` (`feat(invoices): refresh Mombongo invoices from Factures`) adds the one reusable ADMIN invoice-refresh operation (`invoices/invoiceRefresh.ts` + `useInvoiceRefresh.ts`) the Factures screen, harvest-invoice detail, and harvest-invoice pay-status-check all now share — focus/foreground/pull-to-refresh, throttled+deduped, partial-failure-honest, never touches the mutation queue. Replaced `harvest-invoice/[id]/pay.tsx`'s false "updates automatically" promise with a real "Actualiser le statut" action. 55 new/changed tests, 1609/1609 passing; `tsc`/`eslint`/`expo export --platform web` clean.

**QA end-to-end verification**: found and fixed a real bug along the way — `AROM-Backend` `1ec143c` grants `isMombongoWebhook()` read on `harvestOffers` (its create/update-only rule was missing read, so the webhook's own "mark the matching offer won" query failed permission-denied on every first `invoice_issued` delivery; the `harvestInvoices` doc itself still got created first, so no invoice was ever lost, but this affects production too, not just QA). QA Worker redeployed from `a33bda1` (version `c6ff0935-b9bf-4d96-a919-c3f544d4e9bb`) with QA-only Mombongo secrets provisioned (`externalIntegrations/mombongo`, `mombongo-webhook@system.arom.cd` in `arom-qa`); a signed synthetic `invoice_issued` webhook created a `harvestInvoices` doc cleanly (200) and replayed idempotently (no duplicate). Live-app verification of the Factures screen picking up the new row (steps 7–11 of the session's own test plan) could not run — the shared Playwright browser was in use by another session — so that remains a manual/next-session confirmation; the refresh logic itself is covered by the unit/integration tests above.
- **Correction (2026-09-18): "not yet deployed to production Firestore" was NOT a deliberate decision** — it's the same systemic bug as the rest of Sprint 08's backend work: `1ec143c` was never pushed past this machine, and even AROM-Backend's own CI has never once deployed rules to the correct project (`deploy-rules.yml` has hardcoded `--project arom-production` — a different Firebase project from the real `arom-production-657f2` — since its first commit; fix identified, see "⚠️ Production deployment integrity" above). Confirmed directly: the live `arom-production-657f2` rules (fetched 2026-09-18) do not grant `isMombongoWebhook()` read on `harvestOffers`.

**External Mombongo API remains unavailable** (`createExternalInvoice`/`getExternalPublishedListings` both HTTP 500 on their dev host, reconfirmed 2026-09-17 — a regression on their side since 2026-08-31, outside AROM's control). Unrelated to and not blocking the above — invoice reflection depends only on the inbound webhook, which doesn't call Mombongo's API at all.

### 09 — Partner-order pilot — 🚧 Blocked
No approval record found anywhere in AROM-Documentation for any of the 6 explicit gate items (payment/refund ownership, official formats/prices, reservation duration/release policy, minimum customer/delivery data, canonical partner naming). Do not start. Unaffected by, and not advanced by, the Mombongo invoice-reflection work above — that work is limited to `harvestInvoices`, not `partnerOrders` or any customer-order flow.

### 10 — Field validation and release — ⬜ Not started
Depends on 06/07/08/09.

## Native-testing infrastructure (2026-09-17, adjacent to Sprint 10, not a sprint itself)

Sprint 10 ("field validation and release," see `prompts/10-field-validation.md`) requires three **uncoached observation sessions on a real, lowest-end Android device** — impossible without a real installable build. That build didn't exist safely before this work; it does now.

- **Dedicated QA Firebase project `arom-qa`** — own Auth/Firestore/Storage, same `firestore.rules`/`storage.rules`/indexes as production, deployed and verified. A separate QA deployment of `AROM-Production`'s trusted Worker routes runs at `arom-production-qa.purple-hat-3cb3.workers.dev`, built from clean commit `17d2277`, with its own `inventory-service` system account. **Zero data sharing with the live `arom-production-657f2` project** — verified twice (bundle grep, `eas-cli build:inspect`). Synthetic-only seed data (fake producer/reception, real product/price catalogue shape, 100 units of pre-seeded 500ml stock so Orders testing works immediately) and three test accounts (`admin@arom-qa.test`, `production@arom-qa.test`, `commerciale@arom-qa.test`, all `TestPass123!`).
- **`eas.json`'s `qa-apk` profile** (`AROM-Mobile` `d0b4fe9`/`f053d4a`) — internal-distribution Android APK, its own application id `cd.arom.mobile.qa` and display name "AROM QA" (via a small `app.config.js`, only active when `EXPO_PUBLIC_FIREBASE_TARGET=qa`; every other profile is untouched), so it can be installed side-by-side with a production build on the same phone.
- **`docs/qa/ANDROID_QA_CHECKLIST.md`** — real-device checklist covering install/auth, reception, production, quality, orders (explicitly gated on the trusted Worker routes actually being deployed — they are), native behavior (back button, offline, permissions, font scale), defect-recording template.
- **A real QA APK has been built and installed** (EAS build `4d404dfc-cbd3-4344-996d-e37dd59d973b`, commit `f053d4a`) — confirmed package `cd.arom.mobile.qa` throughout the manifest.
- **iOS**: bundle id `cd.arom.mobile`, export-compliance declared (`AROM-Mobile` `93b7e0b`), EAS iOS Distribution Certificate + Provisioning Profile now provisioned (interactive Apple ID step completed by the account owner). `production`-profile build `fbca0b40-2828-4758-9dc5-7ecbd7e0e947` (commit `93b7e0b`, build number 2) is now **FINISHED** (checked 2026-09-18 via `eas build:list --json` — was "in progress" as of 2026-09-17 11:27 UTC). Submission (`eas submit`) and adding TestFlight testers still have not happened.
- **Correction/new finding (2026-09-18): this build cannot reach any real backend for the Sprint 08 trusted-route flows.** `93b7e0b` already has the mobile-side trusted-route rewrite as an ancestor (it calls `fetch` against `${AROM_PRODUCTION_URL}/api/inventory/...`, not direct Firestore writes — confirmed by reading the file content as of that exact commit), but `eas.json`'s `production` build profile sets no `env` block at all, and `eas env:list production` confirms zero EAS-server-side variables are configured for that environment either — unlike `preview`/`qa-apk`, which both explicitly set `EXPO_PUBLIC_AROM_PRODUCTION_URL`. The shipped build's URL almost certainly baked in as the code's own fallback, `http://localhost:8080`. A **new** production build, with this variable configured, will be needed before any of Sprint 08's mobile-side work can function for a real user — independent of and in addition to every other production-deployment gap above.

None of this is Sprint 10 itself — no uncoached session has been run yet. It removes the technical blocker that made Sprint 10 impossible to start.

## Next sprint (corrected 2026-09-18)

The "(a) land the dashboard fix, it's just unmerged" framing above is
**superseded** — `93ccda3`'s content has been confirmed byte-identical on
local `main` (via cherry-pick `a33bda1`) since 2026-09-17. The real
blocker was never the merge; it's that nothing downstream of the merge
ever reached a shared or deployed state. The corrected next step is the
**shared-state and deployment-readiness batch**, in this exact order
(each step gated on the previous one's explicit approval — none of this
should run unattended):

1. **Push all three repos' local `main` to `origin`.** Pure repo hygiene; unblocks real CI on everything written since mid-August.
2. **Merge the two isolated, already-verified deploy fixes**: `AROM-Production`'s `chore/wrangler-deploy-config-fix` (adds `--config wrangler.json` to the deploy step — root-caused and dry-run-verified 2026-09-18) and `AROM-Backend`'s `fix/deploy-rules-wrong-project` (corrects `deploy-rules.yml`'s target from the wrong `arom-production` project to the real `arom-production-657f2` — confirmed via `firebase projects:list` that these are distinct projects with different project numbers).
3. **Deploy `firestore.rules` to the real production project** and diff the result against local `firestore.rules` to confirm it actually matches (don't assume — the live ruleset has drifted from every commit at least once already, via an undocumented manual deploy on 2026-09-16).
4. **Provision (or explicitly decide not to yet) the `inventory-service` Firebase identity and the 3 missing Worker secrets** (`INVENTORY_SERVICE_EMAIL`, `INVENTORY_SERVICE_PASSWORD`, `FIREBASE_WEB_API_KEY`) for `arom-production-657f2`/the production Worker.
5. **Configure `EXPO_PUBLIC_AROM_PRODUCTION_URL` for the `production` EAS profile/environment and cut a new production mobile build** — the currently-installed one cannot reach any backend for these flows regardless of 1–4.

Only once 1–5 are done does it make sense to pick up:

**(a) Sprint 08 Step E (direct sales)** and **Sprint 06's remaining sales-wizard work** — both build on the reservation infrastructure 1–5 would finally make real.

**(b) Sprint 10's first uncoached observation session** — independent of 1–5 (it exercises the currently-working, non-trusted-route parts of the app: reception, production, quality). The technical blocker (a real installable device build) is already cleared. Needs a real worker/tester and the account owner present.

Not sprint 03's remainder (smaller/independent, doesn't block or get blocked by anything else).
Not sprint 09 (explicitly gated, unrelated collection, unaffected by any of the above).

See the 2026-09-18 audit session for the full trigger matrix, compatibility matrix, cutover order, rollback steps, and smoke tests behind steps 1–5.

## Working-tree state (re-verified 2026-09-18)

Every repo's git status (dirty/clean) is unchanged from 2026-09-17 unless
noted; what changed is the **push status**, which was never previously
checked. See "⚠️ Production deployment integrity" above for the full
reasoning.

- **AROM-Mobile**: clean of uncommitted source (only QA screenshot directories `docs/qa/screenshots/`, `qa-screenshots/` untracked — harmless, correctly never committed). `main` at `d034b66` (7 commits ahead of the last tracker update: ADMIN command-centre redesign `c118ff2`/`b637b40`/`86aef83`, nav-highlight fix `660ef9b`, orders cache-namespace fix `bb49c34`, work-queue redesign `cbe18b3`, quality-dedup/finished-stock-honesty fix `d034b66`). **`origin/main` is still `eb9f4a0` (2026-09-01) — 30 commits behind.** No GH Actions deploy exists for this repo either way (its `ci.yml` is verification-only: lint/typecheck/test, no deploy job) — pushing is pure repo hygiene here, zero deploy risk.
- **AROM-Backend**: clean (no dirty files). `main` at `1ec143c` (one commit ahead of the previous note's `02b635d` — the `harvestOffers` webhook-read fix). **`origin/main` is still `d0e7350` (2026-08-20) — 10 commits behind.** Pushing this repo's `main` **would trigger** `deploy-rules.yml` (path-filtered on `firestore.rules`/`firestore.indexes.json`/`storage.rules`) — treat as a production mutation requiring approval, and only after merging `fix/deploy-rules-wrong-project` (see above), or it will deploy to the wrong project yet again.
- **AROM-Production**: `main` at `a33bda1` (confirmed: `93ccda3`'s content is present, byte-identical, via cherry-pick — the old "unmerged" framing above is stale). **`origin/main` is still `c44bcc6` (2026-08-16) — 17 commits behind.** Pushing this repo's `main` **would trigger** `ci.yml`'s `deploy` job unconditionally (no path filter — every push to `main` attempts a Cloudflare Workers deploy) — treat as a production mutation requiring approval, and only after merging `chore/wrangler-deploy-config-fix` (see above), or the deploy will fail exactly as it has 10/10 times before. Still dirty, confirmed unchanged and harmless: `.playwright-mcp/` and ~82 scratch screenshots at repo root. The previously-listed dirty source files (`tasks.ts`, `auth.tsx`, `join.tsx`, `login.tsx`, `storefront/signup.tsx`, `dashboard.tsx`, `routeTree.gen.ts`) and the Mombongo-integration slice are now **all clean/committed** (versioned by `2db292a`) — that part of the prior note is resolved, not stale.
- **AROM-Documentation**: `architecture.md`, `data-model.md`, `flows.md`, `rbac.md`, `roadmap.md`, `runbook.md` still modified, uncommitted — unchanged. **Correction: this file (`PROGRESS.md`) is git-tracked**, not untracked as previously claimed here — `git log` shows 3 prior commits to it (`f548e0d`, `5c818e4`, `b36e629`); it currently shows as modified. The rest of `sprints/Mobile/` (this pack's own `README.md`/`prompts/`, every MOB-01..MOB-11/Sprint DP/PQ/RC/SW folder, and `design-references/`) genuinely is untracked. New since the last note: `sprints/Mobile/design-references/{admin,work-queue,stock-pf}/` — three already-implemented, not-yet-incorporated design kits (see Sprint 07 above). No reception-redesign kit exists anywhere in this repo or AROM-Mobile — checked exhaustively 2026-09-18.
