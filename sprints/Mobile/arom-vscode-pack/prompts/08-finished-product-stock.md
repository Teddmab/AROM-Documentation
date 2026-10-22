# Sprint 08 Finished-product stock and reservations

Read `docs/completion-plan/README.md` first.

Before external orders, implement authoritative finished-product inventory from existing production output, packaging formats, movements, and traceability links. Do not derive sellable stock from raw-material `stockMP`.

The model must support:

- on-hand quantity;
- active reserved quantity;
- available quantity;
- production receipt;
- sale or fulfilment issue;
- damage;
- correction;
- cancellation release;
- lot and product-format traceability.

Use transaction-safe rules so `available = on hand - active reservations` and availability never becomes negative. Add reservation expiry only if the business approves an expiry policy.

Keep an immutable movement trail and actor identity. The user interface shows what can be sold now and what needs attention; it must not hard-code mockup quantities.

Add concurrency tests proving that two devices cannot reserve or sell the same remaining stock.

