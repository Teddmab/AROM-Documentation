# AROM work-queue redesign

## Product decision

`Consignes` is replaced by a useful, role-aware work queue.

- ADMIN label: `À traiter` — decisions, approvals and operational exceptions the administrator can act on.
- Operational-role label: `À faire` — the signed-in poste's next executable actions.
- Secondary tab/section: `En suivi` — informative states only, never mixed into actionable work.

## Core rules

1. An item appears under `À faire`/`À traiter` only if the current account is authorized and the app has a real destination/action.
2. Informational cross-poste records belong under `En suivi`, collapsed by default.
3. Raw email addresses, Firestore stage names and technical IDs are never primary titles.
4. Show human context: producer/cooperative, quantity, product, lot, age and why attention is needed.
5. One dominant CTA per actionable card.
6. Completing an action removes it from the active queue and may place it in `En suivi` or Activity.
7. Empty state appears only when the active queue is genuinely empty.
8. ADMIN's queue is derived from the same canonical priorities used by the ADMIN home and `/admin-priorities`; do not create a second priority engine.
9. Team invitations are ADMIN-only and must use human copy, never leak to Production/Collection.
10. Do not weaken Firestore Rules or rely on client filtering as the security boundary.

## References

- `admin-a-traiter.png`
- `production-a-faire.png`

The mockup values are illustrative. Implementation must use real records and honest empty/loading/offline/error states.
