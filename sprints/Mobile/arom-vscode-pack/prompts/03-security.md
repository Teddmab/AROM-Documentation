# Sprint 03 Staff scoping and Firestore validation

Read `docs/completion-plan/README.md` first.

First produce a read-only report of active staff accounts with a missing, unsupported, or `Personnalisé` poste. Do not mutate any account automatically.

After reporting the migration impact:

1. Require a supported poste for new non-admin mobile access.
2. Propose the safest migration for existing unscoped accounts.
3. Fail closed only after the migration gate is explicitly approved.
4. Add Firestore shape validation for `stockMP`, `clients`, `ventes`, `marketing`, and `charges` using current real fields and the existing `isValidX` pattern.
5. Preserve webhook-only sensitive payment transitions.

Add emulator tests for every role, malformed documents, immutable identifiers, restricted fields, sensitive transitions, and inactive or partner accounts.

Stop before enforcing a breaking rule if real production accounts would lose access without a migration.

