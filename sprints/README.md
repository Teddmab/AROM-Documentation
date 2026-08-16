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
| 01 | [Order → ventes bridge](<[DONE] 01-order-ventes-bridge.md>) | Done |
| 02 | [iOS-style redesign & branding](<[DONE] 02-ios-redesign-branding.md>) | Done |
| 03 | [Bug fixes, responsive audit, landing animations](<[DONE] 03-bugfixes-responsive-animations.md>) | Done |
| 04 | [Account-lockout & signup session fix](<[DONE] 04-account-lockout-fix.md>) | Done |
| 05 | [Admin catalog management](<[DONE] 05-admin-catalog-management.md>) | Done |
| 06 | [Promo banner](<[DONE] 06-promo-banner.md>) | Done |
| 07 | [Storefront order tracking](<[DONE] 07-storefront-order-tracking.md>) | Done |
| 08 | [PawaPay mobile money (stub)](<[DONE] 08-pawapay-payment-stub.md>) | Done — stub phase, real credentials still pending |
| 09 | [Invite-link admin/staff signup](<[DONE] 09-invite-link-admin-staff-signup.md>) | Done |
| 10 | [Google/Facebook/email login](<[DONE] 10-oauth-google-facebook-login.md>) | Done |
| 11 | [Remove Lovable dependency](<[DONE] 11-remove-lovable-dependency.md>) | Done |
| 12 | [Cloudflare deploy pipeline](12-cloudflare-deploy-pipeline.md) | Pipeline built — blocked on Cloudflare credentials |
| 13 | [Guided boutique onboarding](<[DONE] 13-guided-boutique-onboarding.md>) | Done |
| 14 | [Storefront self-service: profile, order & product detail](<[DONE] 14-storefront-self-service.md>) | Done |
| 15 | [WhatsApp order notifications](15-whatsapp-notifications.md) | Todo — from 2026-08-14 kickoff |
| 16 | [Boutique verification (call-confirmation KYC)](<[DONE] 16-boutique-verification.md>) | Done |
| 17 | [Staff poste, per-person bonus tracking, data-level enforcement](<[DONE] 17-staff-poste-data-enforcement.md>) | Done |
| 18 | [Dashboard IA rework, data/logic fixes, KYC review, password reset](<[DONE] 18-dashboard-ia-data-rethink.md>) | Done |
| 19 | [Record detail modals + sidebar sub-nav shortcuts](<[DONE] 19-record-detail-modals-sidebar-shortcuts.md>) | Done |
| 20 | [Parcours production funnel view + responsive modals](<[DONE] 20-parcours-production-funnel.md>) | Done |
