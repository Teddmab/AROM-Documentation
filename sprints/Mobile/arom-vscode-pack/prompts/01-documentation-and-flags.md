# Sprint 01 Documentation and release flags

Read `docs/completion-plan/README.md` first.

Audit the live repositories before editing. Update the mobile architecture, data model, roadmap, and root agent guidance so they describe the as-built system.

Required work:

1. Document AROM Mobile in the system topology.
2. Document `producerInvoices`, `harvestOffers`, and `harvestInvoices` from their real schemas.
3. Mark the historical combined sprint material as superseded where appropriate.
4. Correct the documented Mombongo implementation status.
5. Add a decision record: external customer orders are a separate future capability and must not reuse `producerInvoices` or `mombongoInvoices`.
6. Hide Google and Facebook login actions unless real client IDs are configured.
7. Verify that Mombongo simulation or test flags cannot become enabled accidentally in production.

Do not change business behavior beyond safe feature visibility and configuration guards.

Completion gate:

- documentation links are valid;
- release configuration fails safely;
- typecheck, lint, current tests, and Expo web export pass.

