# MOB-05 — Owner Operational Follow-Up

Implement the six-screen flow in `references/owner-operational.png`.

## Flow

1. Business overview
2. Activity health
3. Yield analysis
4. Production-lot detail
5. Corrective-action assignment
6. Assignment confirmation

## Acceptance

- Mode Administrateur appears throughout.
- Values come from existing authoritative calculations.
- Reception → production → stock traceability is preserved.
- Owner can assign a task to an authorized responsible person.
- Loading, no-data, stale/offline and assignment-failure states exist.
- Tests cover drill-down and assignment authorization.

