# Master Prompt — MOB-01 Role-Aware Login

Implement **MOB-01 — Role-Aware Login** in the existing React Native/Expo application.

Use the attached screenshot and `references/role-login-reference.png` as the visual reference. Read `README.md` completely before changing code.

## Mandatory approach

1. Inspect the existing navigation, authentication, role, theme, form, image and test code.
2. Report the exact files you will reuse or modify.
3. Preserve existing API contracts and working authentication behavior.
4. Implement one shared configurable login component, not five duplicated screens.
5. Pause and report any conflict between the screenshot, sprint specification and existing business rules.

## Role configuration

Support:

- `ADMIN` → “Administrateur”
- `COLLECTION_AGENT` → “Agent de collecte”
- `PRODUCTION` → “Production”
- `STOCK_MANAGER` → “Gestionnaire de stock”
- `SALES` → “Ventes & Boutique”

Reuse the repository’s actual role identifiers when they differ. Do not invent or migrate backend roles in this sprint.

Each configuration provides:

- Display title
- Description
- Role badge
- Hero asset
- Accessible image description
- Icon

## Functional requirements

- Render the chosen role variant.
- DRC `+243` telephone input with normalization and validation.
- Submit through the existing authentication service.
- Continue into the existing OTP/authentication route.
- “Utiliser un autre rôle” returns to role selection.
- “Besoin d’aide ?” uses the existing support route when available.
- Provide loading, validation, offline, server-error and unauthorized-role states.
- Prevent double submission.
- Do not allow the client-selected role to grant authorization.
- After sign-in, reconcile the chosen role with server-authorized roles.

## Visual requirements

- Match the screenshot’s hierarchy, spacing, green/gold/ivory palette and photographic hero.
- Use existing design tokens where compatible; centralize missing tokens.
- Minimum 48 dp touch targets and 54 dp primary button.
- Respect safe areas, keyboard avoidance, Dynamic Type/font scaling and screen readers.
- Optimize and cache images for low-end Android devices.
- Avoid gradients, glassmorphism and excessive pill styling.

## Quality requirements

- Strict TypeScript.
- No hard-coded business data inside generic components.
- Unit/component tests for every role.
- Tests for phone validation, submit loading, offline error, role switching and unauthorized role.
- Run the repository’s typecheck, lint and relevant test commands.

## Final response

Return:

1. Files created or modified
2. Reused existing components/services
3. Behavior implemented
4. Tests and commands executed
5. Deviations from the screenshot and why
6. Remaining blockers

Update `README.md` with implementation evidence. Do not implement dashboards or unrelated authentication changes.

