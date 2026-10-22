# Sprint SW — Réorganisation de l'assistant de vente

> **Status: Done, scaled down from proposal** (implemented 2026-09-02,
> same day as this proposal). Shipped: `WIZARD_STEPS` in
> `features/sale/saleDraft.ts` reordered from
> `["client", "details", "lots", "payment", "verify"]` to
> `["details", "lots", "client", "payment", "verify"]` — products first
> (what's being sold, then which lots), client second, payment last.
> `sale-new.tsx`'s navigation (`goBack`/`goNext` index `WIZARD_STEPS` by
> `indexOf(step)`) and the autosave skip-list (checks step *names*) are
> both already order-independent, so no other navigation logic changed.
> Every step component (`ClientStep`, `DetailsStep`, `LotsStep`,
> `PaymentStep`, `VerifyStep`) is untouched. One real side effect of the
> reorder was fixed: since `details` is now the first step, its
> canal/format/quantité/prix actions needed to start marking the draft
> dirty themselves (previously that only happened via client selection,
> which was step one) — otherwise "Retour" from the first step would
> have silently discarded typed values with no confirm prompt, a
> regression the old order never had.
>
> **Not built: automatic lot allocation** (the proposal's step 3) and
> **client selection becoming conditional** (step 4). Investigating the
> real code before implementing surfaced that both rested on premises
> the schema doesn't support:
> `features/sale/components/LotsStep.tsx`'s own doc comment says a
> production lot has "no 'already sold' concept in this schema" — unlike
> reception→production, which has a real remaining-quantity computation
> (`features/production/productionAvailability.ts`), there is no
> equivalent for production→vente to reuse. Building one would be new,
> unvalidated domain logic with real overselling risk if wrong, not a
> small reuse — deferred to a future sprint that starts by validating
> with SALES staff whether manual lot choice today is a deliberate
> decision or pure friction (this proposal's own open question, still
> unanswered). Separately, `saleValidation.ts`'s `validateClientStep`
> requires `idClient` unconditionally and `ClientStep.tsx`'s empty state
> confirms every sale must be tied to a real client — client selection
> was moved later in the sequence, but never made optional. The "−/+
> quantity controls" and a new visual format/photo picker from the
> proposal were also not built: `DetailsStep.tsx`'s existing
> ChipRow/TextField UI already works and wasn't the audit's actual
> complaint — the complaint was step *order*.
>
> Verified: `bun run typecheck`/`lint`/`test` all pass (110 suites, 992
> tests — `__tests__/SaleWizardScreen.test.tsx` and
> `__tests__/saleDraft.test.ts` updated for the new order).
>
> Original proposal below, written 2026-09-02 from the same external UX
> audit as Sprint RC.

## Objective

Reorder the sale wizard around how a salesperson actually thinks about
a transaction — what's being sold, to whom, for how much — instead of
around the traceability structure (production lots) the app needs
internally but a salesperson shouldn't have to reason about mid-sale.

## Current state

- `app/sale-new.tsx`: `client → détails → lots → paiement → vérifier`
  (`ClientStep` → `DetailsStep` → `LotsStep` → `PaymentStep` →
  `VerifyStep`), same wizard-with-internal-step-state architecture and
  draft lifecycle (explicit save, debounced autosave, concurrent-editor
  lock, stale-reference handling) as `reception-new.tsx`/
  `production-new.tsx` — this proposal doesn't touch that machinery.
- `Vente` records carry `productionIds?: string[]` — a sale's link back
  to the specific production lot(s) it draws from, shown on
  `app/sale/[id].tsx` as a "Lots sources" traceability section
  (read-only, post-sale) — that display is fine and out of scope here;
  the concern is `LotsStep` asking the salesperson to *choose* lots
  during the sale itself.
- No evidence/photo/location/duplicate-detection steps exist on this
  wizard (unlike reception) — nothing to preserve there.

## Proposed redesign

1. **Products first**: format/photo picker (bottle formats already have
   a defined set — 50cl/33cl/30cl per `production/model.ts`'s
   `q500`/`q330`/`q300` fields) instead of opening on client selection.
2. **Quantities** with large `−`/`+` controls, showing real availability
   in plain language (derived from existing stock, not a new
   computation) rather than requiring the salesperson to already know
   which lot has enough left.
3. **Automatic lot allocation**: the system picks eligible production
   lots to satisfy the sale (oldest-first or whatever rule
   `LotsStep`'s current manual-selection logic already effectively
   encodes — reuse that logic's *rule*, just stop asking the human to
   apply it by hand). Manual lot choice becomes a fallback shown only
   when automatic allocation can't fully satisfy the requested quantity
   (split across lots, insufficient stock, etc.) — not removed, just no
   longer the default path.
4. **Client** selection moves after quantities, only when the sale
   actually needs one (matches `ClientStep`'s existing optionality, if
   any — confirm against `features/sale/model.ts`'s `Client`
   optionality before implementation).
5. **Payment**, then a **confirmation** screen showing total, amount
   encaissé, and remaining balance — same fields `PaymentStep`/
   `VerifyStep` already collect and display, just at the end of a
   reordered sequence.

## Constraints preserved

- `productionIds` still gets set correctly on the resulting `Vente` doc
  — automatic allocation must write the same field manual selection
  does today, so `sale/[id].tsx`'s traceability section keeps working
  unchanged.
- The draft/autosave/concurrent-editor-lock lifecycle, unchanged.
- Whatever stock-availability check `LotsStep` currently performs before
  allowing a lot to be selected must still hold — automatic allocation
  is not exempt from the same availability rule a human selecting
  manually is bound by today.

## Out of scope

- Any change to `PaymentStep`'s payment-method handling.
- Any change to what `sale/[id].tsx` displays post-sale.

## Open questions

Carried from the audit's own list, specific to this flow:

- Does SALES normally select specific production lots deliberately
  (e.g. preferring older stock, matching a customer's quality
  preference), or is manual selection today mostly just friction with
  no real decision behind it? This determines whether automatic
  allocation is a genuine simplification or a removal of a choice
  someone actually relies on.
- Is `Client` selection already optional in the current wizard, or does
  every sale require one today? Confirm against
  `features/sale/components/ClientStep.tsx`/`saleValidation.ts` before
  committing to "only when needed" in step 4 above.
- Real usage validation: observe one real sale end-to-end, including at
  least one case where a single lot can't fully cover the requested
  quantity, before committing to the automatic-allocation-first design.
