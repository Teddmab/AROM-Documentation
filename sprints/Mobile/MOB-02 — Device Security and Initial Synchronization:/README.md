# MOB-02 — Device Security and Initial Synchronization

> **Status: Done** (verified 2026-09-01). `src/app/access-setup.tsx`
> implements the 4-step flow (RoleConfirmStep → SecureDeviceStep →
> SyncStep → ReadyStep); `useInitialSync.ts` does real, resumable
> per-category sync. Covered by `SecureDeviceStep.test.tsx`,
> `SyncStep.test.tsx`, `ReadyStep.test.tsx`, `deviceSecurity.test.ts`,
> `pin.test.ts` — all passing, along with the full 732-test suite,
> typecheck and lint.

## Objective

Complete the post-login access flow before opening a role dashboard.

## Flow

1. **Role confirmed**
   - Display the selected, server-authorized role.
   - Allow the user to return and choose another authorized role.

2. **Secure this device**
   - Offer biometric unlock when supported.
   - Always provide a 4–6 digit PIN fallback.
   - Explain briefly that this protects business data.
   - Never confuse the local PIN with the account password.

3. **Initial synchronization**
   - Download the minimum data required for the selected role.
   - Show clear progress and the current data category.
   - Prevent duplicate initialization.
   - Handle lost connection, retry, partial progress and insufficient storage.
   - Never discard already downloaded valid data silently.

4. **Ready**
   - Confirm the active role and last synchronization.
   - Primary action: “Accéder à mon espace”.
   - Route to the correct protected role home.

## Exact French copy

- `Votre rôle`
- `Agent de collecte`
- `Changer de rôle`
- `Sécurisez cet appareil`
- `Utiliser Face ID / l’empreinte`
- `Créer un code PIN`
- `Vos données professionnelles resteront protégées.`
- `Préparation de votre espace`
- `Synchronisation en cours…`
- `Vous pourrez travailler même sans connexion.`
- `Votre espace est prêt`
- `Accéder à mon espace`
- `Réessayer`
- `Continuer plus tard` only when a valid minimal offline dataset already exists.

## Visual rules

Follow the approved AROM login reference: cream background, deep green, warm yellow accent, large controls, short French copy, realistic role image, and no bottom navigation during setup. Show `Mode Agent de collecte` on role-aware screens.

## Security and data rules

- Roles and permissions remain server-authoritative.
- Store tokens and the local PIN/biometric key using the repository’s secure-storage approach.
- Never store the PIN as plain text.
- Initial sync must be resumable and scoped to the active organization and role.
- Do not invent endpoints; adapt to existing services and report missing contracts.

## Acceptance criteria

- Flow starts after role selection and ends at the correct protected home.
- Biometric and PIN fallback work correctly.
- Progress, offline, retry, partial failure and storage-error states exist.
- Back navigation cannot bypass security or role authorization.
- iOS and Android layouts support large text and 48px touch targets.
- Tests cover success, cancellation, retry, role mismatch and direct-route protection.

## Out of scope

Role dashboard implementation, business forms, organization creation, and permission administration.

