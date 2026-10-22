# MOB-02 — Access Setup and Initial Synchronization

> **Superseded.** This six-screen plan (organization selection + role
> selection as separate steps) was replaced by the four-step "Device
> Security and Initial Synchronization" version in
> `MOB-02 — Device Security and Initial Synchronization:/README.md` —
> canonical, verified Done. AROM has no organization/multi-tenancy
> concept, so an organization-selection screen was never applicable; see
> that folder's README and MOB-06A's `RESEARCH_NOTES.md` for why. Kept
> here for history.

## Goal

Implement the six-screen setup flow in `references/access-setup-reference.png`.

## Screens

1. Organization selection
2. Role selection
3. Role-capability explanation
4. Biometric or PIN setup
5. Initial synchronization progress
6. Setup completed

## Rules

- Organizations and authorized roles come from the authenticated server session.
- Role selection never grants permission.
- Users with one organization/role may skip selectors, while retaining a settings route to switch when authorized.
- Biometrics use the platform secure-authentication mechanism; never store raw PINs.
- First synchronization must not report success until required data is stored locally.
- Interrupted sync resumes safely and does not duplicate records.

## Required states

- Loading, empty, retry and server error
- No organizations
- One/multiple organizations
- One/multiple roles
- Role unauthorized/revoked
- Biometrics unavailable/denied
- Sync progress, interrupted, failed and complete
- Insufficient device storage

## Acceptance criteria

- All six screens match the reference hierarchy.
- Back navigation cannot bypass required setup.
- Organization and role choices are validated against server claims.
- Initial sync shows real progress, not a simulated timer.
- Completed setup persists securely.
- iOS/Android, accessibility, small screens and offline recovery are tested.
- Typecheck, lint and tests pass.

## Out of scope

- Role dashboards
- Creating organizations or roles
- Editing permissions
- Background mutation synchronization

