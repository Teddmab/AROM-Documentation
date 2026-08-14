# Sprint 12 — Independent Cloudflare deploy pipeline

**Status:** Pipeline built — [Teddmab/AROM-Production#11](https://github.com/Teddmab/AROM-Production/pull/11) — blocked on `CLOUDFLARE_API_TOKEN` / `CLOUDFLARE_ACCOUNT_ID` repo secrets before it actually deploys

**Rôle concerné :** Infrastructure (n/a — no user-facing role)
**Page / zone :** n/a (CI/deploy pipeline, not a UI change)

## Pourquoi maintenant

Sprint 11 disconnected `AROM-Production` from Lovable, which was the
only thing deploying it. As of that sprint, merging a PR to `main` does
not update the live site — this is the follow-up that fixes that.

## Dans le périmètre

A `deploy` job added to the existing `AROM-Production/.github/workflows/ci.yml`
(not a separate `deploy.yml` — same file as the `verify` job, so deploy
only runs once verification has already passed, via `needs: verify`).
Runs on push to `main` only (not PRs), and does `bun run build` +
`wrangler deploy` via `cloudflare/wrangler-action@v4`, targeting
`.output/server` — the Nitro `cloudflare-module` build output.

A root `wrangler.jsonc` pins the Worker name to `arom-production`
(previously fell back to an auto-generated one, `teddmab-arom-production`).
Nitro auto-detects and merges a root wrangler config into the one it
generates at build time — confirmed locally: after adding the file, the
"Using auto generated worker name" log line disappears and the
generated `.output/server/wrangler.json` shows `"name": "arom-production"`.

## Hors périmètre

Custom domain setup on Cloudflare (can follow once the Worker itself is
deploying reliably). Any change to *what* gets built — Nitro's
`cloudflare-module` preset is unchanged from sprint 11.

## Décisions déjà actées

- **This targets Cloudflare Workers, not Cloudflare Pages** — came up
  directly while setting this up: the Cloudflare dashboard's "connect a
  repo" flow for a new *Pages* project (generic build-command +
  output-directory fields, `*.pages.dev` domain) is the wrong product
  for this build. `cloudflare-module` outputs a Workers entrypoint +
  `wrangler.json`; Pages' generic flow doesn't understand that shape and
  would deploy a static shell with no SSR — the dashboard, storefront
  auth, and PawaPay server functions would all break silently (the
  build would "succeed" and produce a broken site). If deploying from
  the Cloudflare dashboard directly rather than GitHub Actions, the
  equivalent is "Workers Builds," a separate project type from Pages.
- **One workflow file, not two** — `deploy` is a second job inside the
  existing `ci.yml`, gated with `needs: verify`, rather than a
  standalone `deploy.yml`. Keeps the "does this commit pass verification
  before it ships" relationship explicit in one place.
- **`wrangler-action` installs its own `wrangler`** — no need to add it
  as a project devDependency; `npx wrangler` covers manual/local deploys.

## Contraintes

Needs, from whoever owns the Cloudflare account this should deploy
under:
- A Cloudflare **API token** scoped to "Edit Cloudflare Workers".
- The Cloudflare **Account ID**.

Both go into `AROM-Production`'s GitHub repo secrets
(`CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`) — never committed,
never pasted into a chat transcript. See
[runbook.md](runbook.md#frontend-deploys) for exactly where.

## Livrable

`AROM-Production` (`wrangler.jsonc`, `deploy` job in
`.github/workflows/ci.yml`). `AROM-Documentation` (this file,
runbook.md).

## Test de fumée

- [x] `bun run build` locally — generated `.output/server/wrangler.json`
      picks up the pinned name from the root `wrangler.jsonc`
- [ ] Push to `main` triggers the new workflow *(needs the two repo
      secrets first — can't be verified from this session, no
      Cloudflare credentials available here)*
- [ ] Workflow succeeds and the live site reflects the deployed commit
- [ ] `firebase deploy` (AROM-Backend's separate pipeline) is unaffected
- [ ] Confirm the Worker name is stable across redeploys, not
      regenerated each time
