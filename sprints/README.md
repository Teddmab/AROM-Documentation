# Sprints

One file per sprint, one sprint per role + one page/flow — that scoping
rule is deliberate (see [flows.md](../flows.md)): it's what makes the
sequence converge on a complete, working path for each role through each
part of AROM, instead of a pile of disconnected fixes.

Every sprint file follows the same shape (the same fields the
[Sprint Brief generator](https://claude.ai/code/artifact/0a6920c5-dd67-449c-a168-8fed9eea354a)
produces, so a file here can be copy-pasted straight into it to regenerate
the prompt, or a freshly generated prompt can be dropped in here as a new
file once the sprint is planned):

- **Role & page** — who this is for, where it lives
- **Why now** — the real motivation
- **In scope / Out of scope**
- **Decisions already locked** — not up for relitigating
- **Constraints**
- **Deliverable** — which repo(s), roughly which files
- **Smoke test** — always present, always last

Sprints are grouped into milestones. The first one is
[v1-mvp.md](v1-mvp.md) — the complete commerce loop (admin publishes →
partner orders → partner pays → admin fulfills) working end to end for
every role. Numbering is chronological across the whole project, not
restarted per milestone.

| # | Sprint | Status |
| --- | --- | --- |
| 01 | [Order → ventes bridge](01-order-ventes-bridge.md) | Done |
| 02 | [iOS-style redesign & branding](02-ios-redesign-branding.md) | Done |
| 03 | [Bug fixes, responsive audit, landing animations](03-bugfixes-responsive-animations.md) | Done |
| 04 | [Account-lockout & signup session fix](04-account-lockout-fix.md) | Done |
| 05 | [Admin catalog management](05-admin-catalog-management.md) | Done |
| 06 | [Promo banner](06-promo-banner.md) | Done |
| 07 | [Storefront order tracking](07-storefront-order-tracking.md) | Done |
| 08 | [PawaPay mobile money (stub)](08-pawapay-payment-stub.md) | Done — stub phase, real credentials still pending |
| 09 | [Invite-link admin/staff signup](09-invite-link-admin-staff-signup.md) | Done |
| 10 | [Google/Facebook/email login](10-oauth-google-facebook-login.md) | Done |
| 11 | [Remove Lovable dependency](11-remove-lovable-dependency.md) | Done |
| 12 | [Cloudflare deploy pipeline](12-cloudflare-deploy-pipeline.md) | Pipeline built — blocked on Cloudflare credentials |
| 13 | [Guided boutique onboarding](13-guided-boutique-onboarding.md) | Done |
| 14 | [Storefront self-service: profile, order & product detail](14-storefront-self-service.md) | Done |
| 15 | [WhatsApp order notifications](15-whatsapp-notifications.md) | Todo — from 2026-08-14 kickoff |
| 16 | [Boutique verification (call-confirmation KYC)](16-boutique-verification.md) | Todo — from 2026-08-14 kickoff |
