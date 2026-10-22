## Sprint SW — Réorganisation de l'assistant de vente

**Status: Done, scaled down from proposal** (implemented 2026-09-02,
same day as the proposal — from an external UX audit). Shipped the
part of the reorder that was safe to build: `sale-new.tsx`'s wizard now
runs `détails → lots → client → paiement → vérifier` (was
`client → détails → lots → paiement → vérifier`), a single-line reorder
of `WIZARD_STEPS` in `saleDraft.ts` — every step component is unchanged.
Automatic production-lot allocation was **not** built: the real data
model has no "remaining bottles per lot" concept to drive it (unlike
reception→production's `productionAvailability.ts`), so building one
would have been new, unvalidated domain logic with real overselling
risk, not a reorder. Client selection stays fully required, not
conditional — confirmed against real validation. See this folder's
README for what shipped and what's deferred.
