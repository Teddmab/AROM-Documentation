Apply the focused member/finished-stock correction using these supplied files:

Runtime assets:
- assets/images/work-queue/task-new-member.png
- assets/images/stock-pf/finished-stock-flow.png

Design reference and specification:
- docs/design-references/stock-pf/stock-pf-story-empty.png
- docs/design-references/stock-pf/CLAUDE_MEMBER_STOCK_VISUAL_CORRECTION.md

Read the complete Markdown specification before editing.

Important corrections:

1. The new-member action must use its own team/member asset. Never reuse a stock, carton, invoice or bottle illustration for a person-related action.

2. The supplied member illustration is intentionally identity-neutral. Do not use a generated human portrait as though it were the real employee’s likeness. The real name/email from the member record remains authoritative.

3. The screenshot currently renders “Lot PDR_001 en attente de contrôle” twice. Fixing React keys is insufficient. Trace both source records and deduplicate the domain action with a canonical identity such as:
   quality-required:{productionId}

Do not merge genuinely different actions for the same lot—for example, quality-required and quarantine-resolution.

4. Redesign `/stock-pf` so it tells the real lifecycle:

   Contrôle qualité libéré
   → Bouteilles ajoutées
   → Disponibles à la vente

Use `finished-stock-flow.png` as a compact explanatory illustration, not as proof that stock exists.

5. The empty-state reason must be derived honestly:
   - Use “Aucun lot n’a encore été libéré par le contrôle qualité” only when no released lot exists.
   - Otherwise use “Aucune bouteille disponible pour le moment.”

6. Show “Voir les contrôles en attente” only when:
   - pending controls actually exist;
   - the current account is authorized;
   - the CTA has a real working destination.

7. The per-format section must distinguish:
   - en stock/onHand;
   - réservé/reserved;
   - disponible/onHand - reserved.

Do not display fabricated zeroes while data is loading, unavailable, partial or failed.

8. Optimize both runtime assets:
   - preserve transparency;
   - no upscaling;
   - longest edge approximately 384–480px;
   - centralized require map;
   - expo-image with contentFit="contain";
   - decorative and excluded from screen-reader repetition;
   - reduced/hidden on short screens and large font scales.

9. Refresh live QA screenshots for:
   - ADMIN queue with a new member;
   - duplicate quality sources producing one card;
   - no finished stock;
   - available finished stock;
   - some stock reserved;
   - all on-hand stock reserved, leaving zero available;
   - offline and partial failure;
   - 360×640.

Run targeted tests, full Jest, typecheck, lint and Expo export.

Do not deploy production.
Do not start another sprint.
Do not commit unrelated screenshot directories or unused assets.
Return the semantic deduplication identity, asset dimensions after optimization, exact files changed, screenshot paths and final test totals.