# Sprint 09 Partner-order pilot

Read `docs/completion-plan/README.md` first.

Do not implement until these are approved:

- who collects customer money;
- who owns refunds and failed-delivery losses;
- official product formats and effective prices;
- reservation duration and release policy;
- minimum customer and delivery data;
- canonical partner naming.

After approval, create a separate `partnerOrders` domain with:

- signed request verification;
- stable external order ID and idempotency key;
- channel or partner ID;
- minimal customer and delivery data;
- items, currency, totals, and price version;
- independent payment and fulfilment states;
- reservation link;
- timestamps and raw signed payload for audit;
- exactly-once linked `vente` and reconciliation state.

Reuse existing HMAC, webhook, and idempotency patterns. Do not reuse `producerInvoices`.

Reject unknown products, stale prices, invalid signatures, duplicate conflicting payloads, and insufficient stock. Cancellation releases stock once. Fulfilment creates one sale once. ADMIN metrics never count both records as separate sales.

