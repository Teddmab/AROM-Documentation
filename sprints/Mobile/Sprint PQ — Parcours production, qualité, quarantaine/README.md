# Sprint PQ — Parcours production, qualité, quarantaine

> **Status: Proposal, not yet approved for implementation.** Written
> 2026-09-02 from an external UX audit (ChatGPT, given a full
> `AROM-Mobile` screen map): "production and quality follow the database
> structure more closely than the worker's mental model." Bigger in
> scope than Sprint RC — this touches three routes' relationship to each
> other, not one interruption point — so it needs more validation before
> a build commitment (see Open questions).

## Objective

Present production, quality control, and quarantine resolution as one
continuous lot journey instead of three separately-listed record types
a production worker has to already understand are related.

## Current state

- `/productions` (list) → `/production-new` (wizard: source → quantity →
  output → packaging → yield → verify) → `/production/[id]` (detail).
- `/production/[id].tsx` already links forward to quality: "Contrôler ce
  lot" (→ `/quality-control-new`) when no control exists yet, "Voir le
  contrôle qualité" (→ `/quality-control/[id]`) once one does — this
  cross-linking is real and already reasonably good.
- `/quality-controls` (list, filtered to lots needing control) is a
  **separate** entry point a PRODUCTION-role user reaches independently
  from `/production/[id]`'s own link — the same underlying data
  (`getLotsToControl`, `features/quality/qualityRepository.ts`) surfaced
  through two different routes depending on how you arrived.
- Quarantine resolution
  (`app/quality-control/[id]/resolve.tsx`) is architecturally a
  **second, independent** `QualityControl` document
  (`resolvesId` pointing at the original — see that file's own doc
  comment: "never an in-place update of the quarantine doc, which stays
  untouched forever"). The UI reaches it via a "Résoudre la quarantaine"
  button on `quality-control/[id]/index.tsx`, but nothing on screen
  explains *why* resolving creates a new document rather than editing
  the quarantined one — a worker doesn't need to know this to use the
  button, but also can't tell a resolution happened by looking at the
  original record afterward without following the link forward.

## Proposed redesign

1. **One status-driven lot view**, anchored at `/production/[id]`:
   à produire → production enregistrée → contrôle requis → libéré /
   quarantaine / rejeté, with **one dominant next action** matching
   whichever state the lot is actually in (mirrors the "one dominant
   primary action" rule this app's guided-screen contract already
   enforces everywhere else — see e.g. `GuidedPrimaryAction.tsx`'s own
   doc comment).
2. **Quarantine resolution shown as visible history** on that same lot
   view, not just a forward link: once resolved, both the original
   quarantine record and the resolution record appear together
   ("Premier contrôle — En quarantaine" / "Nouveau contrôle — Libéré"),
   so a worker sees the outcome without needing to understand
   `resolvesId` exists.
3. **`/quality-controls` becomes secondary**, not a peer entry point —
   either folded into `/production/[id]`'s own list-adjacent surfacing
   (e.g. the existing "needs attention" affordance already used
   elsewhere in this app — see `adminOperations.ts`'s
   `QUALITY_ATTENTION_PREFIXES`) or kept only as a filtered "what needs
   my attention across all my lots" view, clearly subordinate to the
   per-lot journey rather than a competing way to reach the same data.

## Constraints preserved

- `quality-control/[id]/resolve.tsx`'s core invariant — a quarantine
  document is never edited in place, resolution is always a new
  document with `resolvesId` set — stays exactly as-is. This proposal
  changes how that fact is *presented*, not the underlying write model
  or `firestore.rules`.
- The draft/autosave/concurrent-editor-lock lifecycle on
  `production-new.tsx` and `quality-control-new.tsx` is unaffected —
  this is a navigation/presentation change, not a wizard-internals one.
- The admin soft-gate (`AdminReadOnlyGate`, session-scoped active mode —
  see the mode-switch work already shipped) continues to apply
  identically regardless of how this journey is laid out.

## Out of scope

- Any change to what quality-control decisions mean or how they're
  validated (`features/quality/qualityValidation.ts`).
- Stock movement — the audit raised it as a related open question (see
  below) but it's a separate record type with no natural single-lot
  view of its own; not folded into this proposal.

## Open questions

Carried from the audit's own list, specific to this flow — and the
reason this proposal is marked bigger/less certain than Sprint RC:

- Are stock movements performed by the production worker, a dedicated
  storekeeper, or both? This affects whether stock movements belong
  anywhere near this consolidated lot view or stay fully separate.
- Does folding `/quality-controls` into something secondary lose a
  workflow some production staff actually rely on today (e.g. triaging
  multiple lots' pending controls at once, which a single-lot view
  can't do as well as a list can)?
- Real usage validation, same as the audit's overall recommendation:
  observe one real production → quality-control cycle, including at
  least one quarantine resolution, before committing to folding these
  three routes together.
