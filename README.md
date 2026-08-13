# AROM — Documentation

Documentation for the AROM pineapple-juice production ERP: the operational
dashboard used to run the campagne pilote 2026 (approvisionnement,
production, stock, ventes, marketing, finances, primes) and the partner
storefront where buyers place orders.

## Repos

| Repo | Purpose |
| --- | --- |
| [`Teddmab/AROM-Production`](https://github.com/Teddmab/AROM-Production) | Frontend — TanStack Start + React app (dashboard, storefront, login). Deployed via Lovable to Cloudflare Workers. |
| [`Teddmab/AROM-Backend`](https://github.com/Teddmab/AROM-Backend) | Firebase config — Firestore/Storage security rules, admin provisioning scripts, CI to deploy rules. |
| `Teddmab/AROM-Documentation` (this repo) | Architecture, data model, RBAC design, operational runbook. |

## Start here

- **[architecture.md](architecture.md)** — deep analysis: what existed before this work, what changed, and why. Read this first.
- **[rbac.md](rbac.md)** — the admin/staff/partner role model, how it's enforced, and how to manage accounts.
- **[data-model.md](data-model.md)** — Firestore collections and the fields they carry.
- **[runbook.md](runbook.md)** — day-to-day operations: creating accounts, seeding data, deploying rules, local dev.
- **[roadmap.md](roadmap.md)** — known gaps and suggested next steps, roughly in priority order.
