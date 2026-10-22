## MOB-09 — Real Mombongo payment gateway

**Status: Mostly done** (verified 2026-09-01) — the gateway call and
`"pending"` result are real; the waiting step is **not** actually
listener-driven yet despite this entry's own acceptance criterion below
— see this folder's README for the specific gap.

Revised 2026-08-31: no checkout redirect exists — the real contract
returns a Stripe `clientSecret` (embedded via Stripe SDK) for card and
requires phone+operator in-app for mobile money (an on-device prompt, not
a URL). Implements `MombongoProducerPaymentGateway` against MOB-06B's
existing `ProducerPaymentGateway` interface (`mode: "real"`), waiting step
driven by a Firestore listener, not polling.
