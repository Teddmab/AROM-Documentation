# Sprint RC — Réception et création de producteur

> **Status: Done** (implemented 2026-09-02, same day as this proposal).
> `producteur-new.tsx` derives a `minimal` mode from the loaded draft's
> own `receptionDraftId` (not the raw URL param — correctly survives
> resuming a draft later, e.g. from `producteur/[id].tsx`'s "Reprendre
> le brouillon"), which reuses the existing, unmodified
> `receptionDraftId` hand-off `reception-new.tsx` already sent — no new
> signal, no new screen. Minimal mode: `ContactStep.tsx` hides
> secteur/territoire, `ActivityStep.tsx` hides capacité/prix (both
> already-optional per `producteurValidation.ts`, unchanged), the photo
> step is skipped entirely via a local `MINIMAL_WIZARD_STEPS` sequence,
> and `ProducteurReviewStep.tsx` never offers to edit a photo this flow
> never asked for. GPS capture deliberately stays available even in
> minimal mode — a single tap, not a form field, and the agent is
> standing at the collection point right now. 19 tests in
> `ProducteurWizardScreen.test.tsx` (7 minimal-mode, 2 full-mode
> regression, rest pre-existing and unaffected); `typecheck`/`lint`/
> `test` all pass (110 suites, 991 tests).
>
> Original proposal below, written 2026-09-02 from an external UX audit
> (ChatGPT, given a full `AROM-Mobile` screen map) that identified this
> as the most field-critical flow worth redesigning — a real field agent
> uses it daily, unlike most admin-only screens this app also has. Not a
> from-scratch redesign: the audit's own verdict on the app overall was
> "solid operational foundation... would not redesign it from scratch,"
> and this proposal keeps that spirit — it reworks one interruption
> point in an otherwise-working wizard, not the wizard itself.

## Objective

Reduce the friction of the single most common interruption in the
reception flow: an agent recording a delivery discovers the producer
isn't in AROM's system yet and has to create one mid-reception. Today
that means leaving the reception wizard for a full, separate
producer-creation flow before the fruit already sitting at the
collection point gets recorded at all.

## Current state

- `app/receptions.tsx` → `app/reception-new.tsx`
  (producteur → détails → qualité → prix → preuve → vérifier, one route
  with internal step state, MOB-04/MOB-04A's draft-lifecycle
  architecture: explicit save, debounced autosave, a concurrent-editor
  lock, stale-reference/storage-failure handling).
- The producer step itself
  (`features/reception/components/ProducerStep.tsx`) already has a
  "Producteurs récents" section (recently-transacted-with producers,
  from `getRecentProducteurIds` in `receptionRepository.ts`) above the
  full searchable list — this part already matches good practice and
  doesn't need to change.
- When no match exists, "Ajouter un producteur" routes to
  `app/producteur-new.tsx` (MOB-04B) — a full guided creation flow
  ending in its own result screen, from which "Utiliser pour cette
  réception" hands control back to the reception wizard with the new
  producer selected. This hand-back mechanic is already the right
  shape; the audit's concern is with what `producteur-new.tsx` *asks
  for* along the way, not the hand-back itself.
- `Producteur` (`features/reception/model.ts:17-46`) has required-in-UI
  fields for name, village, secteur, territoire, telephone, produit,
  capaciteKgMois, prixConvenu, statut, observations — the full profile a
  standalone "manage my producers" flow legitimately wants, but more
  than what's needed in the moment fruit is waiting to be weighed.

## Proposed redesign

Not a new screen — a **shorter path through the existing
`producteur-new.tsx` wizard specifically when it's launched from inside
a reception** (it already receives a `returnTo`/context param for the
hand-back; extend that to also mean "minimal mode"):

1. Collect only: nom, village, téléphone (if available), produit —
   the fields the reception step itself already needs to display for
   the producer it's about to record fruit from.
2. Skip capaciteKgMois/prixConvenu/statut/observations/photo in this
   path — those stay required for the standalone "Nouveau producteur"
   entry point (from `/producteurs` or Agent home), where there's no
   fruit waiting and completeness matters more than speed.
3. On save, hand back to the reception wizard immediately with the new
   producer selected — same mechanic as today, just reached sooner.
4. The full profile fields become editable later from the producer's
   own detail screen (`app/producteur/[id].tsx`) at a moment that isn't
   time-pressured — not dropped, deferred.

## Constraints preserved

- The existing draft/autosave/concurrent-editor-lock lifecycle on both
  `reception-new.tsx` and `producteur-new.tsx` — this proposal changes
  which fields are asked for, not the save/sync mechanics.
- Duplicate detection (`features/reception/duplicateDetection.ts`) stays
  exactly as-is.
- Offline-first: minimal-mode creation must queue and sync the same way
  full creation does today — no new online-only requirement.

## Out of scope

- Any change to the "Producteurs récents" section, which the audit
  didn't flag as a problem.
- Any change to the standalone producer-creation entry points (list,
  Agent home) — minimal mode is reception-context-only.
- Firestore schema changes — `Producteur`'s optional fields already
  support being unset; no new field needed to make this work.

## Open questions

Carried from the audit's own list, specific to this flow:

- How often do agents actually interrupt `/reception-new` to create a
  producteur? If this is rare in practice, the priority of this sprint
  should drop relative to others.
- Which reception wizard step causes the most abandoned/saved drafts
  today — is producer creation actually the friction point, or is it
  elsewhere in the flow (evidence capture, price entry)?
- Do agents commonly share one device, and if so, does a
  minimal-then-complete-later producer record create confusion about
  who's responsible for finishing the profile?

The audit's own recommendation stands: observe one real reception
end-to-end, without coaching, before committing to this redesign over
a smaller fix (e.g. just reordering `producteur-new.tsx`'s existing
steps so the required-for-reception fields come first).
