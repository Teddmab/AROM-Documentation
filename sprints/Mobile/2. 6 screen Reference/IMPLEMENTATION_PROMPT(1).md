# Master Prompt — MOB-02 Access Setup

Implement **MOB-02 — Access Setup and Initial Synchronization** in the existing React Native/Expo application.

Use the attached screenshot and `references/access-setup-reference.png` as the visual reference. Read `README.md` first.

## Before editing

Inspect existing authentication, organization, role, secure-storage, biometric, local-database, synchronization, navigation and test code. List the exact files to reuse or modify. Do not invent endpoints or replace working services.

## Implement

- Organization selector using server-authorized organizations.
- Role selector using server-authorized roles.
- Role-capability explanation driven by configuration.
- Biometric/PIN setup using existing secure platform facilities.
- Initial data synchronization with real progress and retry.
- Completion screen showing active campaign and pending tasks when available.
- Safe resume after interruption.
- Loading, empty, failure, permission, offline and storage-limit states.

## Security

- Client selection never grants access.
- Revalidate organization and role against server claims.
- Do not store raw PINs, tokens or sensitive data in AsyncStorage.
- Clear incomplete sensitive setup state safely on account change.

## Engineering

- Reuse the MOB-01 design system/components.
- Strict TypeScript; no duplicated screens.
- Respect safe areas, keyboard, font scaling, screen readers and 48 dp targets.
- Do not simulate synchronization progress.
- Add tests for every navigation branch, failure state and authorization rule.

## Final response

Return modified files, reused services, implemented behavior, tests run, screenshot deviations and blockers. Update the sprint README with implementation evidence. Do not implement dashboards.

