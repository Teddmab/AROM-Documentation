# MOB-04A — Reception Draft Lifecycle

## Objective

Allow an Agent de collecte to deliberately save an unfinished reception, find it later, resume from the exact step, and delete it safely.

## Core behavior

### Save a draft

- Every reception wizard step exposes `Enregistrer comme brouillon`.
- Continue auto-saving locally after meaningful field changes, but keep the explicit action visible and understandable.
- Saving a draft must not enqueue a Firestore reception write.
- A draft can be saved with incomplete data; validate only data already entered.
- Store the current step, completed fields, local evidence references, created/updated timestamps, uid, organization and active role/poste scope.
- Show confirmation: `Brouillon enregistré` and `Vous pourrez le reprendre plus tard.`
- Actions: `Continuer maintenant` and `Quitter`.

### Access drafts

- Add a `Brouillons` filter/card to `Mes réceptions`.
- Show draft count on the Agent home only when greater than zero.
- Draft cards show available producer/material information, `Étape X sur 6`, last modification and local-only status.
- Never mix drafts belonging to another uid, organization or role.

### Resume

- Opening a draft shows a summary with `Reprendre le brouillon`.
- Resume at the last valid incomplete step, with every saved value restored.
- If reference data changed or is no longer accessible, retain the draft and explain what must be reselected.
- Prevent two simultaneous editors for the same draft on one device.

### Delete

- Destructive confirmation: `Supprimer ce brouillon ?`
- Explain: `Les informations non envoyées seront définitivement supprimées de cet appareil.`
- Require explicit `Supprimer`; cancel leaves the draft untouched.
- Clean up local photos/location evidence owned only by that draft.

### Submit conversion

- Final submission reuses the draft's stable client ID.
- Atomically mark the draft as submitting before enqueueing.
- On successful server acknowledgement, remove the editable draft and expose the reception detail.
- On offline/pending/failed submission, preserve the immutable queued record; do not recreate an editable duplicate.
- Double taps and app restarts must never create two receptions.

## Navigation

- Agent home → `Brouillons` → drafts list.
- `Mes réceptions` → `Brouillons` filter.
- Wizard secondary action → save confirmation.
- Back/close with unsaved changes offers: `Enregistrer le brouillon`, `Continuer`, `Abandonner`.

## Required states

- No drafts
- Draft saved
- Auto-save in progress/failed
- Draft list loading
- Draft reference data unavailable
- Local evidence missing
- Resume after app restart
- Delete confirmation/success/failure
- Draft currently being submitted
- Storage unavailable or full

## Data rules

- Extend the existing MOB-02/MOB-04 local store; do not add a parallel draft store.
- Use a versioned draft schema and migration path.
- Drafts are local-first and are not counted as received stock, expenses, tasks or completed receptions.
- Encrypt/protect local sensitive data using the repository's existing device-security approach where applicable.
- Never store duplicated image bytes when a stable local file reference is sufficient.

## Visual rules

- Persistent `Mode Agent de collecte` chip.
- Yellow/cream status treatment for drafts; never use green synchronized styling.
- Short French copy, clear file/edit/trash icons with text, large targets.
- Follow the attached AROM reception reference.

## Acceptance criteria

- User can explicitly save from every wizard step.
- Draft survives app restart and offline use.
- User can find drafts from home and reception history.
- Resume restores the exact step and entered values.
- Delete requires confirmation and removes draft-owned evidence.
- Submission converts one draft into exactly one queued/synchronized reception.
- Drafts never affect operational totals before submission.
- Existing MOB-01–MOB-04 checks remain green.

## Tests

Cover explicit save, auto-save, incomplete values, restart recovery, scope isolation, list/filter/count, resume step restoration, changed reference data, delete cleanup, storage failure, double submit and draft-to-queued conversion.

