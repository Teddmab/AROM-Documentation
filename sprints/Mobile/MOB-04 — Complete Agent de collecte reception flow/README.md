# MOB-04 — Agent de collecte: Complete Reception Flow

> **Status: Done** (verified 2026-09-01), including the 04B producer-creation
> extension. Full reception wizard, offline mutation queue with stable
> IDs, idempotent sync, draft lifecycle, and guided producer creation with
> duplicate detection are all real — see
> `src/features/reception/{receptionRepository,receptionSync,receptionDraft,receptionValidation}.ts`
> and `src/features/producteur/*`. Extensive test coverage
> (`receptionSync.test.ts`, `duplicateDetection.test.ts`,
> `ReceptionWizardScreen.test.tsx`, etc.) passes alongside the full suite.

## Objective

Complete the Agent de collecte vertical slice from the existing role home to a safely saved and synchronized raw-material reception.

## User journey

1. **Mes réceptions**
   - Real reception history, newest first.
   - Search/filter only where existing data supports it.
   - Status: `Brouillon`, `En attente`, `Synchronisation`, `Synchronisé`, `Échec`.
   - Primary action: `Nouvelle réception`.

2. **Select producer**
   - Search the locally synchronized producteurs dataset.
   - Show name plus an existing reliable identifier/location field.
   - Recent producers may be shown from real local history.
   - Do not create visit schedules or producer fields that do not exist.

3. **Reception details**
   - Select the real material/product type supported by the current schema.
   - Enter received quantity and unit.
   - Use large numeric controls and the device numeric keyboard.
   - Apply existing validation and precision rules.

4. **Quality check**
   - Show only quality attributes supported by the current backend model.
   - Use image-led, explicit choices for non-technical users.
   - If no structured quality fields exist, record only the existing note/photo evidence; do not invent server fields.

5. **Price and costs**
   - Unit price, calculated subtotal, transport and supported additional costs.
   - Reuse existing server/domain calculation rules.
   - Display FC amounts clearly with grouped digits.
   - Do not let UI totals become authoritative when the backend calculates them.

6. **Evidence**
   - Optional/required photo according to existing rules.
   - GPS only with explicit permission and only if the schema supports location.
   - Compress photos for low bandwidth while preserving readable evidence.
   - Explain why each permission is requested.

7. **Verification**
   - One readable summary: producer, material, quantity, quality, prices/costs, evidence and total.
   - `Modifier` links return to the relevant step without losing data.
   - Primary action: `Enregistrer la réception`.

8. **Confirmation and sync**
   - Online: save once, then show synchronized receipt/detail.
   - Offline: store locally with a stable client-generated ID and show `Enregistrée sur cet appareil`.
   - Queue automatic retry when connectivity returns.
   - Manual retry for failed items.
   - Never silently discard or duplicate a reception.

9. **Reception detail**
   - Show all confirmed fields, local/server ID mapping, creation time and sync state.
   - Allow draft editing/deletion before submission.
   - After synchronization, expose edit/cancel only if current permissions and backend contracts support them.

## Navigation

- Home `Nouvelle réception` and central `+` route to the wizard.
- Home reception counts and `Mes réceptions` route to the history list.
- Successful save routes to the reception detail, not directly back to an empty home.
- Android back and iOS gestures must warn before abandoning unsaved changes.

## Offline write architecture

Extend the MOB-02 offline foundation instead of creating a second store:

- Stable client-generated reception ID.
- Draft persisted after each completed step.
- Mutation envelope with entity, operation, payload, uid, role/poste scope, createdAt, attempt count and state.
- States: pending → syncing → synchronized or failed.
- Firestore creation must be idempotent; reuse the stable document ID where compatible.
- Keep local data until server acknowledgement is confirmed.
- Prevent double taps and duplicate queue entries.
- Resume after app restart.
- If the remote schema cannot support safe idempotency, stop and document the blocker rather than shipping duplicate-prone writes.

## Permissions and security

- Only authorized Agent de collecte/poste users can create receptions.
- Revalidate permission before local enqueue and before remote write.
- Scope producer search, drafts and history to the authenticated account/organization and permitted poste.
- Never trust totals, producer IDs or role claims supplied only by the UI.
- Do not log photos, coordinates, tokens or sensitive producer data.

## Visual and accessibility

- Follow the attached AROM reference: cream, deep green, yellow status accent.
- Persistent chip: `Mode Agent de collecte` on every screen and modal.
- Short French instructions, clear icon + text, large fields and sticky primary action.
- One decision per wizard step; visible progress indicator.
- WCAG-conscious contrast, screen-reader labels, large-text support and 48px targets.
- Native camera, photo-picker, permission and numeric-keyboard behavior on iOS/Android.

## Required states

- Loading and empty history
- No matching producer
- Offline producer cache available/unavailable
- Validation error
- Camera/location permission denied
- Photo upload pending/failed
- Draft saved
- Submission in progress
- Local save successful
- Sync failed/retry
- Possible duplicate detected
- Session/permission expired

## Testing

Cover:

- History and detail from real repositories/cache
- Step validation and back/edit navigation
- Calculation adapters against existing domain rules
- Draft persistence and restart resume
- Online submission
- Offline enqueue and reconnect retry
- Stable-ID idempotency and duplicate-tap protection
- Partial evidence upload failure
- Unauthorized direct navigation/write
- Accessibility labels and large text

Run typecheck, lint, all tests and web export. Perform a manual Expo Go/simulator pass when available, especially camera, location, keyboard, offline restart and reconnect.

## Acceptance criteria

- Agent can complete the full reception flow online or offline.
- No invented backend fields or fake values are shipped.
- A saved offline reception survives restart and syncs exactly once.
- Every screen shows `Mode Agent de collecte`.
- History/detail reflect the new local or synchronized record immediately.
- Existing MOB-01 to MOB-03 behavior and tests remain green.

## Out of scope

Visit scheduling, production, stock movement, payments, administrator approvals, generalized sync centre, and any backend field not already supported.

