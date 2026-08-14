# Sprint 03 — Bug fixes, responsive audit, landing animations

**Status:** Done — [Teddmab/AROM-Production#3](https://github.com/Teddmab/AROM-Production/pull/3), [#4](https://github.com/Teddmab/AROM-Production/pull/4)

**Rôle concerné :** Visiteur / Partenaire
**Page / zone :** Landing, Login, Signup

## Pourquoi maintenant

Three separate asks landed together: the login↔home navigation had a
real gap, the copy read as AI-generated (stray em-dashes), and the
landing page had never been checked against real viewport widths or
given any motion.

## Dans le périmètre

Fix the login/home navigation gap (plus a real React anti-pattern bug —
`navigate()` called during render — that surfaced while investigating
it). Replace visible em-dashes across the app with plain punctuation.
Fix the responsive bugs a real viewport audit (360/480/768/1024/1440px)
actually found. Add reversible scroll animations (count-up numbers,
section reveals) to the landing page.

## Hors périmètre

Dashboard, storefront catalog features, anything not on the
landing/login/signup surface.

## Décisions déjà actées

- Landing nav collapsed to a single "Tableau de bord" CTA — `/dashboard`
  already redirects unauthenticated visitors to `/login`, which already
  links to signup for partners, so three competing links were redundant.
- Animations respect `prefers-reduced-motion` and reverse on scroll-up,
  not just play once on load.

## Contraintes

Same as sprint 01.

## Livrable

`AROM-Production` only.

## Test de fumée

- [x] From the landing page, reach `/login` and back without a dead end
- [x] No console warnings on sign-in
- [x] Viewport audit script: zero horizontal overflow at 360/480/768/1024/1440px
- [x] Count-up/reveal animations verified programmatically to reverse on scroll-up
