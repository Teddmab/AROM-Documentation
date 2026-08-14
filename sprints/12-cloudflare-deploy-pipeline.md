# Sprint 12 — Independent Cloudflare deploy pipeline

**Status:** Todo — blocked on a Cloudflare API token + Account ID

**Rôle concerné :** Infrastructure (n/a — no user-facing role)
**Page / zone :** n/a (CI/deploy pipeline, not a UI change)

## Pourquoi maintenant

Sprint 11 disconnected `AROM-Production` from Lovable, which was the
only thing deploying it. As of that sprint, merging a PR to `main` does
not update the live site — this is the follow-up that fixes that.

## Dans le périmètre

A `wrangler.json`/`wrangler.toml` in `AROM-Production` pinning a stable
Worker name (currently falls back to an auto-generated one,
`teddmab-arom-production`, when unset). A GitHub Actions workflow that
builds and runs `wrangler deploy` on push to `main`, mirroring how
`AROM-Backend/.github/workflows/deploy-rules.yml` already deploys
Firestore/Storage rules — same shape, different target.

## Hors périmètre

Custom domain setup on Cloudflare (can follow once the Worker itself is
deploying reliably). Any change to *what* gets built — Nitro's
`cloudflare-module` preset is unchanged from sprint 11.

## Décisions déjà actées

None yet — the account/token question is what's blocking this sprint
from starting for real.

## Contraintes

Needs, from whoever owns the Cloudflare account this should deploy
under:
- A Cloudflare **API token** with Workers Scripts: Edit permission.
- The Cloudflare **Account ID**.

Both go into `AROM-Production`'s GitHub repo secrets
(`CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`) — never committed,
never pasted into a chat transcript.

## Livrable

`AROM-Production` (`wrangler.json`/`.toml`, new
`.github/workflows/deploy.yml`). `AROM-Documentation` (runbook.md,
architecture.md topology once this is live).

## Test de fumée

- [ ] Push to `main` triggers the new workflow
- [ ] Workflow succeeds and the live site reflects the deployed commit
      (check something small and identifiable, e.g. a copy change)
- [ ] `firebase deploy` (AROM-Backend's separate pipeline) is unaffected
- [ ] Confirm the Worker name is stable across redeploys, not
      regenerated each time
