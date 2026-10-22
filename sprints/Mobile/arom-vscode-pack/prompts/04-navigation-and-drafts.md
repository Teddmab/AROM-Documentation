# Sprint 04 Navigation active mode and drafts

Read `docs/completion-plan/README.md` first.

Centralize typed route helpers and owning-mode metadata. The authenticated application state owns `activeMode`; a route parameter may request a destination but must never silently change the mode.

Required behavior:

- replace admin soft-gate wording with `Passer en mode ...`;
- keep the signed-in user as the audit identity;
- Back inside a wizard moves to the previous step;
- leaving the first step offers `Enregistrer le brouillon` or `Quitter`;
- hardware Back, header Back, and tab changes use the same guard;
- a payment waiting screen explains that leaving is safe when confirmation continues independently;
- completed workflows navigate to an explicit destination, never generic `router.back()`;
- producer creation launched from reception returns to the same draft with the producer selected.

Add route, deep-link, Android Back, dirty-state, mode-switch, and duplicate-submit tests.

