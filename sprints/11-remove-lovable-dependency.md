# Sprint 11 — Remove Lovable dependency

**Status:** Done — [Teddmab/AROM-Production#9](https://github.com/Teddmab/AROM-Production/pull/9)

**Rôle concerné :** Infrastructure (n/a — no user-facing role)
**Page / zone :** Toutes les pages (build pipeline, not a UI change)

## Pourquoi maintenant

Requested directly: disconnect the project from Lovable entirely, both
the GitHub integration (on Lovable's own dashboard) and every piece of
Lovable-specific code.

## Dans le périmètre

`vite.config.ts` rewritten without `@lovable.dev/vite-tanstack-config`
(read the wrapper's actual source rather than guessing what it
configured). Removed: the MCP tool server
(`@lovable.dev/mcp-js`, by explicit choice — not a hosting dependency,
just decided not to carry it forward), `lovable-error-reporting.ts`,
`.lovable/`, `AGENTS.md`. Fixed two asset URLs that only resolved
through Lovable's own hosting (both were already 404ing, one of them
the exact bug reported earlier this session).

## Hors périmètre

The actual Cloudflare deploy pipeline replacement — see
[sprint 12](12-cloudflare-deploy-pipeline.md). This sprint only makes
the app deploy-target-agnostic; it doesn't deploy it anywhere.

## Décisions déjà actées

- MCP tool server: removed entirely, not rewritten without the Lovable
  SDK — a deliberate scope choice, made explicitly rather than defaulted.
- Every peer dependency `@lovable.dev/vite-tanstack-config` needed
  (`@tanstack/react-start`, `@tailwindcss/vite`, `vite-tsconfig-paths`,
  `nitro`, `@vitejs/plugin-react`) was already a direct dependency of
  this project — confirmed by reading the wrapper's own `package.json`,
  not assumed. No new packages were added.
- Nitro's `cloudflare-module` preset is still the build target — this
  sprint doesn't change *what* the app builds for, only *how* the Vite
  config gets there and *who* deploys the result (nobody, yet).

## Contraintes

Same as prior sprints (bun, emulator-first testing, PR against `main`).
This one specifically: verify with the actual build artifact, not just
`tsc --noEmit` — a broken Vite/Nitro config would typecheck fine and
still fail to produce a deployable Worker.

## Livrable

`AROM-Production` only (`vite.config.ts`, `package.json`,
`eslint.config.js`, deletions, two asset-URL fixes).
`AROM-Documentation` (architecture.md, runbook.md, roadmap.md).

## Test de fumée

- [x] `bun install` removes the two Lovable packages, adds zero new ones
- [x] `bunx tsc --noEmit`, `bun run lint` both pass
- [x] `bun run build` produces the same `cloudflare-module` Nitro preset
      output as before, all routes present, no leftover MCP routes
- [x] `bun run dev` still serves correctly
- [x] Full regression suite (landing page, admin/partner login,
      order→ventes bridge, catalog management, invite signup,
      account-lockout fix) — 9/9, zero failures
- [x] The `arom-hero.png` 404 reported earlier this session is
      confirmed fixed (`curl -I /logo-hero.png` → 200)
