# MOB-04B — Create and Verify Producers

## Objective

Allow an authorized Agent de collecte to create a missing producer online or offline, use that producer immediately in a reception, and preserve administrator oversight.

## Entry points

- Producer-selection screen: `Ajouter un producteur`.
- Empty search result: `Aucun producteur trouvé` → `Créer ce producteur`.
- Agent home/central `+`: `Nouveau producteur`, only when authorized.
- Producer list: primary action `Ajouter un producteur`.

## Guided creation flow

### 1. Identity

- Choose `Personne` or `Coopérative` only if the existing schema supports both.
- Required display name using canonical existing fields.
- Optional representative name for a cooperative only when supported.
- Normalize whitespace/case without changing the displayed name.

### 2. Contact and location

- Phone using existing format expectations.
- Village/locality and existing address fields.
- GPS with explicit permission and only when supported by the producer model.
- Explain that location helps the team find the producer later.

### 3. Activity

- Select the supplied material/product using existing catalog data.
- Allow multiple products only if the current model supports it.
- Do not invent farm size, expected volume, visit schedule or KYC fields.

### 4. Photo and evidence

- Optional producer/cooperative photo unless existing rules require it.
- Use respectful, medium-wide photography guidance.
- Compress locally for low bandwidth.
- Keep local evidence attached to the producer draft/queue record until acknowledgement.

### 5. Duplicate check

Before submission, compare normalized:

- Phone
- Name/cooperative name
- Village/location
- GPS proximity only if existing coordinates and policy support it

Show `Producteur similaire trouvé` with actions:

- `Utiliser ce producteur`
- `Vérifier les informations`
- `Continuer la création` only for authorized non-exact matches, with an audit reason if the backend supports it.

Block exact confirmed duplicates. When offline, check the local synchronized dataset and clearly mark that server verification is pending.

### 6. Review and creation

- Show identity, contact, location, activity and photo.
- `Modifier` returns to the relevant step without losing data.
- Primary action: `Créer le producteur`.
- Prevent duplicate taps.

### 7. Result

Online: show `Producteur créé` with real sync/verification status.

Offline: show `Enregistré sur cet appareil` and `La vérification se terminera après synchronisation.`

Actions:

- `Utiliser pour cette réception` when launched from the reception wizard.
- `Voir le producteur`.
- `Créer une réception` from other entry points.

## Producer list and detail

- Add a real producer list/search screen if no reusable one exists.
- Local pending producers appear immediately with `En attente de synchronisation`.
- Detail shows only supported fields, creation source/time and honest sync/verification state.
- Agent may edit/delete a local unsubmitted producer.
- After synchronization, editing/deactivation follows existing permissions; do not give broad management rights.

## Offline dependency model

- Use a stable client-generated producer ID compatible with Firestore document IDs.
- Extend the existing mutation queue; do not create another queue/store.
- A reception created with a local producer references the same stable ID.
- Queue dependency: producer create must succeed before dependent reception create.
- Retry safely after restart and connectivity changes.
- Keep dependent receptions blocked—not failed or discarded—while producer creation is pending.
- If producer sync permanently fails, show the reason and allow correction without losing the reception draft.
- Server/local acknowledgement must not create a second producer record.

## Verification and authorization

- Inspect `firestore.rules`, producer services and schema before editing.
- Agent creation must be explicitly allowed and restricted to safe fields.
- If rules currently prohibit creation, implement the smallest reviewed rule/backend change; do not bypass rules or use admin credentials.
- Administrator verification should reuse an existing status/workflow. If none exists, document the contract gap before adding a new status model.
- A newly created producer may be used immediately in the agent's own reception, but must not silently gain organization-wide verified status.
- Record existing audit fields such as creator uid/poste/timestamps where supported.

## Draft behavior

- Producer creation itself supports local auto-save and explicit `Enregistrer comme brouillon`.
- Producer drafts remain separate from submitted pending producers.
- When launched inside a reception, preserve the reception draft while creating/selecting the producer.
- Returning from producer creation restores the reception wizard and selects the new producer.

## Required states

- Loading and empty producer list
- No search result
- Validation errors
- Phone/location/photo permission denied
- Exact and possible duplicate
- Offline duplicate check
- Producer draft saved/resumed
- Local producer pending sync
- Sync failed/retry
- Dependent reception waiting
- Unauthorized creation
- Server schema/rules rejection

## Visual and accessibility

- Persistent `Mode Agent de collecte` chip on every screen/modal.
- One decision per step, short French copy, progress indicator and sticky main action.
- Cream/green AROM design; yellow for pending/review; red only for blocking/destructive states.
- Large touch targets, screen-reader labels, large-text support and native permission behavior.

## Acceptance criteria

- Agent can create a missing producer from the reception flow online or offline.
- New producer is immediately selectable without losing the reception draft.
- Duplicate detection prioritizes using the existing producer.
- Offline producer and dependent reception synchronize in the correct order exactly once.
- Producer creation cannot assign unsafe fields or self-verify.
- Local drafts/pending producers survive restart.
- Existing MOB-01–MOB-04A behavior remains green.

## Tests and verification

Cover entry points, validation, duplicate normalization, exact/possible matches, permission rules, offline draft, stable IDs, queue dependencies, restart/retry, producer-to-reception restoration, unauthorized writes and evidence cleanup. Run typecheck, lint, all tests, Firestore rules tests, web export, and a manual Expo Go pass when available.

## Out of scope

Administrator verification UI, bulk imports, producer payments, visit scheduling, farm inspections and unrestricted synchronized-record editing.

