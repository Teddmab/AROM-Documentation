# Sprint 02 — iOS-style redesign & branding

**Status:** Done — [Teddmab/AROM-Production#2](https://github.com/Teddmab/AROM-Production/pull/2)

**Rôle concerné :** Tous les rôles (surface visuelle)
**Page / zone :** Landing, Login, Signup, Storefront

## Pourquoi maintenant

A real AROM logo became available (replacing the Lovable-hosted
placeholder asset), and the customer-facing surfaces needed a more
native, modern feel.

## Dans le périmètre

The AROM logo everywhere it was referenced (landing, login, signup,
storefront, dashboard nav, PDF export header) plus a real
`favicon.ico`/`apple-touch-icon.png`. iOS-derived UI patterns on the
storefront/landing/auth pages: translucent blurred sticky nav bars with
safe-area insets, grouped inset-list cards, a +/- stepper for cart
quantities, a sticky bottom cart bar, capsule buttons with press
feedback, squircle app-icon framing for the logo.

## Hors périmètre

The dashboard (internal ERP tool) — only its logo reference was swapped,
no layout changes.

## Décisions déjà actées

- Fonts (Fraunces + Plus Jakarta Sans) were confirmed identical to the
  live Lovable deployment already — no font change was needed or made.
- Colors unchanged — this was a layout/interaction restyle, not a
  rebrand.

## Contraintes

Same as sprint 01.

## Livrable

`AROM-Production` only.

## Test de fumée

- [x] Landing, login, signup, storefront catalog/cart/orders, and
      dashboard nav visually verified at desktop + iPhone viewport
      against the Firebase emulator
