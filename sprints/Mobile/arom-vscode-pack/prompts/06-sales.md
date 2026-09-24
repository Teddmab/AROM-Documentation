# Sprint 06 Product-first sales

Read `docs/completion-plan/README.md` first.

Reorder the sale wizard:

1. products;
2. quantities;
3. optional customer;
4. payment;
5. review.

Use real catalogue data and authoritative finished-product availability. Provide large minus and plus controls. Do not show production-lot IDs during normal sale entry.

Allocate eligible production lots automatically using existing traceability rules. Ask the user only if automatic allocation cannot succeed. Confirm total, amount paid, and remaining balance.

Preserve existing sales rules, prevent double taps, and create each sale once through stable idempotency.

If finished-product availability is not authoritative, do not estimate it. Keep external ordering disabled and record the dependency on Sprint 08.

Test allocation, insufficient stock, offline entry, partial payment, cancellation, duplicate submission, and accessibility.

