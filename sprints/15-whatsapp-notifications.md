# Sprint 15 — WhatsApp order notifications

**Status:** Todo

**Rôle concerné :** Partenaire (reçoit) / Admin (déclenche indirectement)
**Page / zone :** Storefront (mes commandes), Dashboard — Commercialisation

## Pourquoi maintenant

Requested at the 2026-08-14 kickoff (client: Ethical Mine): partners
currently only find out about order/delivery status changes by opening
the app. Most partners communicate over WhatsApp already, and the
client explicitly asked for status updates to reach them there.

## Dans le périmètre

A WhatsApp message sent to the partner's `phone` (collected at
onboarding, sprint 13) when their order's `status` changes to
`confirmed` (include the `deliveryDate` if set, per sprint 07) or
`fulfilled`. Content: order number, new status, delivery date if known.

## Hors périmètre

- Two-way WhatsApp (replying to change/cancel an order) — notification
  only, one direction.
- Notifications for every field change — only the two transitions above.
- SMS/email fallback for partners without WhatsApp — not raised at
  kickoff; revisit if it comes up.
- Admin-side notifications (e.g. "new order received") — the client's
  ask was specifically about partners hearing from AROM, not the
  reverse; admin already sees new orders live via `onSnapshot` in the
  dashboard.

## Décisions déjà actées

None yet — this needs the same kind of decision PawaPay (sprint 08)
already went through:

- **Provider.** WhatsApp Business API access needs either Meta's Cloud
  API directly (requires a verified WhatsApp Business Account) or a
  provider like Twilio/360dialog that wraps it. No credentials exist
  yet for either — this sprint should ship a **stub phase** first,
  matching sprint 08's pattern: build the send-on-status-change trigger
  and message composition against a stub that logs what *would* be
  sent, so the UI/flow is ready and swapping in a real provider later
  is a small, isolated change.
- **Trigger mechanism.** Firestore has no server-side "on write" trigger
  without Cloud Functions (deliberately deferred — see roadmap
  Infrastructure #10). Two options once real credentials exist:
  client-side (whoever's dashboard session performs the status change
  calls the send, same client-triggered pattern as the PawaPay stub) or
  Cloud Functions (reliable regardless of who's online, but crosses the
  "no Cloud Functions yet" line). Needs a decision before the real
  (non-stub) integration — flag for the client.

## Contraintes

Same "no Cloud Functions yet" constraint as the rest of the project. A
partner's `phone` must exist and be well-formed (already validated at
onboarding, sprint 13) before a send is attempted.

## Livrable

`AROM-Production` (status-change hook in `OrdersCard`'s `setStatus`/
`confirmWithDeliveryDate`/`fulfillAndConvert`, a stub WhatsApp-send
function mirroring `AROM-Production/src/lib/payments/pawapay.ts`'s
shape). `AROM-Documentation` (data-model.md, flows.md — provider
decision once made).

## Test de fumée

- [ ] Admin confirms an order with a delivery date — a stub "would send"
      log/toast shows the composed message and target phone number
- [ ] Admin marks an order fulfilled — same, for the fulfilled message
- [ ] An order with no phone on the partner's profile (shouldn't happen
      post-sprint-13, but legacy accounts might lack it) doesn't crash
      the status-change flow — the notification attempt fails silently
      or logs, the status change itself still succeeds
