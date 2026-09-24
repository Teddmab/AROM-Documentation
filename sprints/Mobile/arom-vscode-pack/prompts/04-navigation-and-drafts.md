# Sprint 04 Navigation and drafts

Read `docs/completion-plan/README.md` first.

> **Superseded in part (2026-09-24).** Active modes and mode switching no longer exist: access is capability-based (`docs/access/CAPABILITIES.md`) and every account shares one structure (`docs/simplification/UNIFIED_APP_AUDIT.md`). Ignore the mode items that used to be here; the Back, draft and deep-link items below still apply.

Centralize typed route helpers. A route parameter may request a destination but never grants or changes access — the destination re-derives it from the signed-in account.

Required behavior:

- keep the signed-in user as the audit identity;
- Back inside a wizard moves to the previous step;
- leaving the first step offers `Enregistrer le brouillon` or `Quitter`;
- hardware Back, header Back, and tab changes use the same guard;
- a payment waiting screen explains that leaving is safe when confirmation continues independently;
- completed workflows navigate to an explicit destination, never generic `router.back()`;
- producer creation launched from reception returns to the same draft with the producer selected.

Add route, deep-link, Android Back, dirty-state, and duplicate-submit tests.

