# MOB-01 — Role-Aware Login

> **Superseded (2026-08-27).** The role-first, phone/OTP flow this sprint
> describes and the implementation evidence below were replaced the same
> day by a login-first flow (email/password + Google/Facebook, real
> Firebase auth) at the user's explicit request — role selection now
> happens *after* login and shows only the roles the authenticated
> account is actually authorized for. See `AROM-Mobile`'s `src/app/login.tsx`,
> `role-select.tsx`, and `src/features/auth/roleAuthorization.ts` for the
> current flow. This file is kept for history; treat its "Scope"/"States"/
> "Implementation evidence" sections below as describing a flow that no
> longer exists in the codebase.

## Goal

Implement the five AROM role-aware login variants shown in `references/role-login-reference.png` for React Native/Expo on iOS and Android.

## Roles

- Administrateur
- Agent de collecte
- Production
- Gestionnaire de stock
- Ventes & Boutique

## Scope

1. Role selection entry or preselected role route.
2. Shared login screen with role-specific photo, title, description and icon.
3. DRC telephone input (`+243`) with country selector.
4. Continue to the existing authentication/OTP flow.
5. “Utiliser un autre rôle” returns to role selection.
6. “Besoin d’aide ?” opens the existing help/support route or a documented placeholder.
7. Loading, invalid-number, offline and server-error states.
8. Keyboard, safe-area, accessibility and small-screen handling.

## Security rule

Role selection changes presentation only. Never grant permissions from the selected role. After authentication, use server-authorized roles. If the selected role is unauthorized, explain this and offer an authorized role.

## Required reusable components

- `RoleLoginScreen`
- `RoleHero`
- `RoleBadge`
- `PhoneNumberField`
- `PrimaryButton`
- `OfflineHint`
- Role configuration map; no duplicated screen implementations

## States

- Default
- Focused phone input
- Invalid number
- Submitting
- Offline with existing session
- Offline without existing session
- Authentication failure
- Selected role unauthorized

## Acceptance criteria

- All five variants use one shared component and data configuration.
- Visual hierarchy matches the reference.
- Role label is visible on every variant.
- Phone validation accepts supported DRC formats and normalizes before submission.
- Existing authentication contracts remain unchanged.
- No role privilege is granted client-side.
- Minimum touch target is 48 dp; primary button is at least 54 dp.
- Screen works with font scaling, keyboard open, small Android devices and iOS safe areas.
- Tests cover all roles, validation, navigation, offline behavior and unauthorized roles.

## Out of scope

- Backend authentication redesign
- Creating new server roles
- Role dashboards
- Registration and invitation workflows

## Definition of done

- iOS and Android verified.
- Tests, lint and typecheck pass.
- No duplicated role screens.
- Sprint documentation updated with actual files and commands.

## Implementation evidence (2026-08-27)

**Scope conflicts surfaced and resolved before implementation** (per this
sprint's "pause and report" instruction):

1. No React Native/Expo app existed anywhere in the AROM workspace to
   implement "into." Resolved by creating a new sibling repo,
   `AROM-Mobile`, scaffolded with `bunx create-expo-app` (Expo Router +
   TypeScript, Expo SDK 57 / React Native 0.86 / React 19.2). Not yet
   pushed to a GitHub remote — local only until the user creates one.
2. No phone/OTP authentication backend exists — AROM's Firebase project
   only has email/password and Google/Facebook auth for staff (see
   `AROM-Documentation/rbac.md`), no phone field on staff accounts. Real
   Firebase Phone Auth would need console changes plus a staff data-model
   change, both out of scope here ("Backend authentication redesign").
   Resolved by building the full UI/states against a mocked `AuthService`
   interface (`src/features/auth/authService.ts` /
   `src/features/auth/mockAuthService.ts`) so a real implementation can
   drop in later with no UI change. The five role IDs (`ADMIN`,
   `COLLECTION_AGENT`, `PRODUCTION`, `STOCK_MANAGER`, `SALES`) are
   presentational-only, mobile-app-local identifiers for the same reason —
   reconciling them with real server-authorized roles is MOB-02's job.
3. Hero photography for the five roles doesn't exist as standalone assets.
   Resolved by cropping the five photos directly out of this sprint's own
   `logins.png` reference (`AROM-Mobile/assets/roles/*.png`) as temporary
   placeholders — swap for real AROM staff photography when supplied.

**Files** (all in the new `AROM-Mobile` repo):

- `src/features/auth/roleConfig.ts` — the five role definitions (title,
  description, icon, hero image, badge) plus `RoleId`/`RoleConfig` types.
- `src/features/auth/authService.ts`, `mockAuthService.ts` — the auth
  seam and its mock implementation (`AuthFailureError`,
  `UnauthorizedRoleError`).
- `src/features/auth/phone.ts` — DRC (+243) phone validation/normalization.
- `src/features/auth/sessionCache.ts` — minimal on-device "has this phone
  logged in before" record, just enough to distinguish the two offline
  states (real local persistence is MOB-02's "initial synchronization").
- `src/features/auth/useAuthFlow.ts` — the state machine (idle/submitting/
  failure/unauthorized-role) driving the screen, wired to
  `@react-native-community/netinfo` for the offline states.
- `src/features/auth/components/` — `RoleLoginScreen` (the single shared,
  configurable screen required by the spec), `RoleHero`, `RoleBadge`,
  `PhoneNumberField`, `PrimaryButton`, `OfflineHint`.
- `src/theme/tokens.ts` — colors/fonts/radius/spacing, converted from
  `AROM-Production/src/styles.css`'s OKLCH tokens to RN-usable hex; fonts
  loaded via `@expo-google-fonts/fraunces` and
  `@expo-google-fonts/plus-jakarta-sans` to match the web app's Fraunces +
  Plus Jakarta Sans pairing.
- `src/app/index.tsx` — role-selection entry grid.
- `src/app/login/[role].tsx` — renders `RoleLoginScreen` for a role id
  (also usable as a preselected-role deep link).
- `src/app/otp/[role].tsx` — a minimal, explicitly-temporary OTP
  verification screen (the sprint's "continue to the existing
  authentication/OTP flow" — there is no existing one; see gap #2 above).
- `src/app/home.tsx` — placeholder post-auth screen proving the flow
  completes; MOB-02/MOB-03+ replace it.

**Behavior implemented:** all 8 listed states (default, focused, invalid,
submitting, offline×2, failure, unauthorized-role); phone validation/
normalization; "Utiliser un autre rôle" and "Besoin d'aide ?" (placeholder
alert, no real support route exists yet); no double-submission (button
disables while submitting); role selection never grants a role — the
mock's `UnauthorizedRoleError` path is exercised and shows the
server-authorized alternative instead of proceeding with the picked one.

**Tests and commands run** (from `AROM-Mobile/`):
- `bun run typecheck` (`tsc --noEmit`) — pass.
- `bun run lint` (`expo lint`) — pass.
- `bun run test` (Jest via `jest-expo` + `@testing-library/react-native`) —
  34 tests pass across `phone.test.ts`, `roleConfig.test.ts`,
  `RoleLoginScreen.test.tsx` (all 5 roles, validation, submit/loading,
  failure, unauthorized-role + its offered alternative, navigation, both
  offline states, minimum touch-target size).
- Manual verification via `expo start --web` + a headless-browser pass:
  role grid → all 5 role screens → invalid-phone state → OTP → home
  placeholder, plus the unauthorized-role banner and a **live** offline
  banner (real browser offline event, not just a mocked NetInfo value) —
  all matched the reference and produced zero console errors.
- **Not run**: an iOS/Android simulator pass — this sandboxed environment
  has no simulator/device available. Recommend the user run
  `bun run ios` / `bun run android` before merging, per the sprint's
  "iOS and Android verified" definition-of-done item.

**Deviations from the reference:** none in layout/hierarchy/palette. The
"country selector" chevron in the reference is rendered as a static,
non-interactive DRC flag/+243 chip — AROM only operates in DRC, so a
functional multi-country picker wasn't built (would be first speculative
scope beyond what's needed).

**Remaining blockers for a real (non-mocked) version:** enabling Firebase
Phone Auth in the console, and adding a phone-number field to staff
`users/{uid}` docs (or an equivalent lookup) in `AROM-Backend` — both
explicit, deliberate decisions for the user to make later, not done here.

