# AROM ADMIN overview reference

## Product decision

ADMIN is not an operational role that must repeatedly switch into Collection, Production or Sales just to inspect or initiate work. The ADMIN home is a cross-functional command centre.

- Never show `Passer en mode…` or `Changer de rôle` on the ADMIN home.
- Keep backend authorization and audit identity unchanged.
- ADMIN sees exceptions, operational health and the complete fruit-to-sale chain.
- A deliberate mode may still exist internally for genuinely role-owned write wizards, but it must not be presented as a routine prerequisite for ordinary ADMIN navigation.
- Prefer an ADMIN-safe action or read-only drill-down. If a workflow truly requires a poste-specific context, make that transition explicit only at the workflow boundary, not on the dashboard.

## Reference assets

- `admin-overview-command-center.png`: complete screen reference.
- `admin-pineapple-crate.png`: reception/raw fruit.
- `admin-production-tank.png`: transformation.
- `admin-juice-bottle.png`: finished product.
- `admin-reserved-carton.png`: reserved order stock.
- `admin-invoice-market.png`: invoices/marketplace sales.

These are visual references, not authoritative data. Every number in production must come from the real repositories and preserve existing empty, offline, inconsistent and error states.

## Required information order

1. Identity, notifications and synchronization.
2. Exceptions requiring attention.
3. Fruit-to-sale operational chain.
4. Today’s activity.
5. Operational health.
6. Link to the complete operations directory.

## Accessibility and performance

- Text and status icons remain authoritative; illustrations are decorative.
- Never rely on color alone.
- Use 48dp touch targets and readable French.
- Hide or reduce decorative assets on short screens and large font scales.
- Optimize transparent PNGs and render them with `expo-image`, `contentFit="contain"`.
