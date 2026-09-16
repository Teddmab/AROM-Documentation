# Manual-entry and non-technical-user usability audit (AROM-Mobile)

**Read-only audit. No application code was changed to produce this document.**
Scope: every data-entry screen in AROM-Mobile across ADMIN, COLLECTION,
PRODUCTION, SALES, and the shared authentication/profile/settings flows,
plus a dedicated visual-quantity-feedback review requested alongside this
audit. Compiled 2026-09-16 from a direct reading of `src/app/` and
`src/features/**` (draft/model/validation/sync files) — not from the
Excel workbook or the original sprint specs.

**The mandatory test applied to every screen:** *Can a new employee
complete this screen correctly after a two-minute explanation, without
knowing AROM's database structure, remembering codes, or understanding
technical terminology?*

---

## 1. How to read the tables

Each field is classified as exactly one of:

| # | Class | Meaning |
|---|---|---|
| 1 | Known reference data | A value that already exists somewhere (a producer, a format, a poste) — should be picked, never typed/remembered |
| 2 | Derived/calculated value | Computable from other fields already on record — should never be asked of the user |
| 3 | Authenticated context | Who's using the device, right now — should never be re-typed |
| 4 | Administrator-configured default | A suggested value an admin set — must stay visibly editable, never silently promoted to "the physical fact" |
| 5 | Physical fact requiring confirmation | Something only the person standing there can know (a weight, a visual check) — needs an explicit, confirmable entry |
| 6 | Exceptional override requiring explanation | A deliberate departure from the expected value — needs a plain-language reason |
| 7 | Genuine narrative/free text | A real note, comment, or brand-new master-data record — free text is correct here |

Columns: **Route** · **Role** · **Field** · **Current control** · **Current
source** · **Authoritative source** · **Typed manually?** · **Class** ·
**Recommended behavior** · **Offline** · **Override** · **Validation** ·
**Priority** (High/Med/Low — cost of getting it wrong × how often it's hit).

---

## 2. COLLECTION — Reception wizard (`src/app/reception-new.tsx`)

Six steps: producer → details → quality → price → evidence → verify.

| Field | Current control | Current source | Authoritative source | Manual? | Class | Recommended | Offline | Override | Priority |
|---|---|---|---|---|---|---|---|---|---|
| Producteur | `SelectionCard` list + search | Firestore-synced cache (`getProducteursWithPending`) | `producteurs` collection | No (picked) | 1 | Keep — already correct | Cached list, works offline | "Ajouter un nouveau producteur" link | — |
| Matière (produit) | Read-only inherited card | The producer's own `produit` string, set once at producer creation | The producer doc | No | 2 | Keep — already correct | — | — | — |
| Quantité reçue (kg) | Free numeric `TextField` | Typed | The scale, right now | Yes | 5 | Keep typed; add contextual visual (§7) | Yes | — | High (feeds price, yield, everything downstream) |
| Quantité commandée (kg) | Free numeric `TextField`, optional | Typed | An order/agreement, if any | Yes | 5/1‑ish | Keep typed (no order record exists to pick from yet — see §9.3) | Yes | — | Low |
| Qualité (Conforme/À vérifier/Rejeté) | 3-card radio selector | Tapped | The inspector's own judgment | No (selected) | 5 | Keep — already correct | Yes | — | Med |
| Qualité — note | Free-text, optional | Typed | — | Yes | 7 | **Fix**: currently silently dropped at sync — either wire it into the payload or remove the field (see §9.1, high-risk item) | Yes | — | Med (wasted effort today) |
| Prix par kg (FC) | Free numeric `TextField` | Typed | The agreed price | Yes | 5 (should partly be 4) | Pre-fill from the producer's own `prixConvenu`, if set, editable | Yes | Always editable | High |
| Transport / Autres frais (FC) | Free numeric `TextField`s, optional | Typed | The actual cost | Yes | 5 | Keep typed | Yes | — | Low |
| Valeur/coût total | Read-only computed preview | Derived | — | No | 2 | Keep — already correct | — | — | — |
| Photo | Camera/gallery capture | Captured | — | No | 7-ish (evidence) | Keep | Uploads after sync, non-blocking | — | Low |
| Position GPS | One-tap capture, then read-only | Captured, but **never persisted** (schema has no field) | — | No | — | Either wire it to a real field or remove the capture step — currently pure effort with no payoff (see §9.1) | Local-only | — | Med |
| Numéro (reçu) | Not shown to the user at all | Auto-generated (`NNN_YYYY`), cosmetic | — | No | 2 | Keep — already correct, good pattern | — | — | — |
| Date | Not shown | Auto-stamped at draft creation | — | No | 3 | Keep — already correct | — | — | — |
| Agent (identité) | Not shown | Auth/profile context | — | No | 3 | Keep — already correct | — | — | — |

## 3. COLLECTION — Producer creation (`src/app/producteur-new.tsx`)

Identity → contact → activity → photo → review, plus a duplicate-detection
interstitial. Runs standalone or "minimal" mid-reception (fewer fields).

| Field | Current control | Current source | Manual? | Class | Recommended | Priority |
|---|---|---|---|---|---|---|
| Nom | Free-text, required | Typed | Yes | 7 | Keep — this is genuinely new master data | — |
| Téléphone | Free-text (phone pad) | Typed | Yes | 7 | Keep | — |
| Village | Free-text, required | Typed | Yes | 7 | Keep, but see §9.3 (duplicated free-text village strings across producers with no canonical list) | Med |
| Secteur / Territoire | Free-text, optional, hidden in minimal mode | Typed | Yes | 7 | Keep | Low |
| Produit fourni | Free-text, placeholder **"Ananas"** | Typed, **no catalog exists anywhere in the app** | Yes | Should be **1**, is currently forced into 7 | **This is the single highest-value fix in the whole audit** — see §9.1 #1 | **High** |
| Capacité mensuelle / Prix convenu | Free numeric, optional, hidden in minimal mode | Typed | Yes | 5 | Keep | Low |
| Position GPS | One-tap capture | Captured, **never persisted** (no schema field) | No | — | Same as reception — wire it or remove it | Med |
| Photo | Camera/gallery, optional, skipped in minimal mode | Captured | No | 7-ish | Keep | Low |
| Duplicate match (name+village) | "Utiliser ce producteur existant" / "Continuer" | System-detected | No | — | Keep — good pattern, prevents the exact duplication risk described in §9.3 | — |

## 4. PRODUCTION — Lot creation (`src/app/production-new.tsx`)

Source → quantity → output → packaging → yield → verify.

| Field | Current control | Current source | Manual? | Class | Recommended | Priority |
|---|---|---|---|---|---|---|
| Source (réception) | Card list, computed remaining kg per source, flags over-consumption | Firestore-synced cache | No (picked) | 1 | Keep; add a search box (ProducerStep has one, this list doesn't — could get long) | Med |
| Produit / fournisseur / date | Read-only inherited card | Selected source | No | 2 | Keep | — |
| Kg à transformer | Free numeric | Typed | Yes | 5 | Keep typed; add contextual visual (§7) | High |
| "Restera disponible" | Read-only computed | Derived | No | 2 | Keep | — |
| Jus obtenu (L) | Free numeric, with "Suggéré: X L" one-tap-fill card | Typed, admin-yield-standard-suggested | Yes (with suggestion) | 4→5 | Keep — genuinely correct pattern already (suggestion never silently applied); add contextual visual (§7) | High |
| Explication de l'écart | Conditionally-required free text | Typed | Yes | 6 | Keep — correct pattern (only asked when it's actually needed) | Med |
| Format principal | 3-way pill selector (50/33/30 cl) | Tapped | No | 1 | Keep — already correct | — |
| Quantité de bouteilles | Free numeric, with "Suggéré: N × format" one-tap-fill | Typed, suggestion-assisted | Yes (with suggestion) | 4→5 | Keep; add contextual visual (§7) | High |
| Reste à conditionner | Read-only computed | Derived | No | 2 | Keep | — |
| Rejets (bouteilles) | Free numeric | Typed | Yes | 5 | Keep; consider a separate "rejected" visual (§7) rather than folding it into the packaging count | Med |
| Conformes | Read-only computed | Derived | No | 2 | Keep — correct, matches "never ask workers to calculate percentages manually" | — |
| Lot (référence) | Not shown | Auto-generated (`NNN_AROM`) | No | 2 | Keep | — |
| Responsable | Not shown | Auth/profile context | No | 3 | Keep | — |
| Date | Not shown | Auto-stamped | No | 3 | Keep | — |

## 5. PRODUCTION — Quality control (`src/app/quality-control-new.tsx`) and resolution (`.../resolve.tsx`)

Sample → aspect → quantities → decision → verify (fresh control); a single
dense screen for resolving a quarantine.

| Field | Current control | Current source | Manual? | Class | Recommended | Priority |
|---|---|---|---|---|---|---|
| Lot / date / volume attendu | Read-only inherited card | Denormalized from the production record | No | 2 | Keep | — |
| Échantillon (L) | Free numeric | Typed | Yes | 5 | Keep typed; add the sample-container visual — **this is the item explicitly specified in §7/§8** | **High** |
| Couleur normale / Odeur normale / Bouteille propre | 3× Oui/Non toggle pairs | Tapped | No | 5 | Keep — already correct | — |
| Photo (aspect evidence) | Camera/gallery, optional | Captured | No | 7-ish | Keep | Low |
| Quantité conforme (L) / Quantité rejetée (L) | Free numeric ×2, live cross-check against expected total | Typed | Yes | 5 | Keep; the rejected quantity is the clearest candidate for the "separate, clearly labelled rejected-volume visual" (§7) | Med |
| Décision (Libérer/Quarantaine/Rejeter) | 3-card selector | Tapped | No | 5 | Keep | — |
| Motif | Chip selector, conditionally required | Tapped | No | 6 | Keep | — |
| Commentaire | Free text, optional | Typed | Yes | 7 | Keep | — |
| Responsable / date | Not shown | Auth/profile context, auto-stamped | No | 3 | Keep | — |
| **Resolve screen**: decision re-entry | 2-card selector (Libérer/Rejeter only) | Tapped | No | 5 | Keep | — |
| **Resolve screen**: motif/commentaire | Same as above, conditional | Typed/tapped | Yes | 6/7 | Keep | — |
| **Resolve screen**: original measurements | Read-only, carried forward unedited | Denormalized | No | 2 | Keep — correct, avoids re-asking | — |

**Note on `resolve.tsx`**: this is the densest single non-wizard screen
found in the app — a read-only summary, a decision selector, a
conditional motif picker, a free-text box, and a confirm-modal, all on
one scroll. It's materially the same shape as the 5-step fresh-control
wizard, just collapsed. See §9.4.

## 6. STOCK — Movement wizard (`src/app/stock-movement-new.tsx`)

Type → product → sources → quantity → verify. Coded under the `PRODUCTION`
role (no distinct STOCK role exists).

| Field | Current control | Current source | Manual? | Class | Recommended | Priority |
|---|---|---|---|---|---|---|
| Type (Entrée/Sortie/Ajustement) | 3-card radio selector, icon + hint | Tapped | No | 1 | Keep | — |
| Produit | Free-text `TextField`, placeholder "Ananas" | Typed, **no list exists** | Yes | Should be 1, forced into 7 | Same fix as producer creation's `produit` field — one canonical source, see §9.1 #1 | **High** |
| Unité | Free-text `TextField`, placeholder "Pièce" | Typed | Yes | Should be 1 | Fix alongside `produit` — a tiny fixed set (kg, L, pièce/bouteille) | Med |
| Sources (réceptions/lots) | Multiselect checkbox rows | Firestore-synced cache | No (picked) | 1 | Keep; note there's no computed "remaining/usable" flag on a lot the way reception sourcing has — worth adding for consistency | Low |
| Quantité entrée / sortie | Free numeric `TextField` (typed, not a stepper) | Typed | Yes | 5 | Consider a stepper for small adjustment counts, keep typed entry for large quantities; add contextual visual (§7 — crates/bottles increasing or decreasing) | Med |
| Coût unitaire (FC), optionnel | Free numeric | Typed | Yes | 5 | Keep | Low |
| Observation, optionnel | Free text | Typed | Yes | 7 | Keep | — |
| Linked source (detail screen) | **Raw Firestore document ID shown as the link text** | — | No | — | **Fix**: show a friendly label (lot date/number), not the raw ID (§9.2) | Med |

## 7. STOCK — Finished-goods view (`src/app/stock-pf.tsx`)

Read-only, no data entry at all — explicitly scoped that way in the code's
own doc comment. Each lot row shows `Lot {productionId}` — same raw-ID
issue as the stock-movement detail screen (§9.2). No fix needed beyond
that; nothing to redesign here (§14 item 8).

## 8. SALES — Sale wizard (`src/app/sale-new.tsx`)

Details → lots → client → payment → verify.

| Field | Current control | Current source | Manual? | Class | Recommended | Priority |
|---|---|---|---|---|---|---|
| Canal | Chip row (Hôtel/Restaurant/Boutique/Supermarché/Grossiste/Particulier) | Tapped | No | 1 | Keep | — |
| Format | Plain text chips (500/330/300 ml) | Tapped | No | 1 | Upgrade to illustrated cards (bottle silhouettes) to match the visual-catalogue recommendation in §7/§12 — the audit's "avoid dense forms, prefer illustrated cards for 2–6 choices" rule applies directly here | Med |
| Quantité | Free numeric (no stepper) | Typed | Yes | 5 | Add a stepper — 2–6 bottles is a typical sale, a stepper beats typing for this range | Med |
| Prix unitaire (FC) | Free numeric, **pre-filled from campaign parametres**, overridable | Typed, admin-default-assisted | Yes (with default) | 4 | Keep — already correct (deliberate never-clobber-a-manual-override pattern) | — |
| Client | Search + tappable list, single-select | Firestore-synced cache | No (picked) | 1 | Keep; **no client-creation flow exists on mobile** — flagged as a real gap, not a redesign target | Med |
| Lots sources | Multiselect checkbox rows | Firestore-synced cache | No (picked) | 1 | Keep as traceability; note this is *not* a quantity-per-lot allocation and doesn't check/deduct availability — that's Step C+ reservation work, explicitly out of scope here | — |
| Remise (FC), optionnel | Free numeric | Typed | Yes | 5/6 | Keep | Low |
| Montant encaissé (FC), optionnel | Free numeric | Typed | Yes | 5 | Keep; **no payment-method or payment-status field exists in this wizard at all** — flagged, not a redesign target on its own | Med |
| Total | Read-only live preview | Derived, never itself submitted | No | 2 | Keep — correct pattern | — |
| Numéro | Not shown | Auto-generated | No | 2 | Keep | — |
| Commerciale / date | Not shown | Auth/profile context, auto-stamped | No | 3 | Keep | — |
| Linked source (detail screen) | **Raw Firestore document ID shown as link text** | — | No | — | Same fix as stock movement (§9.2) | Med |

## 9. SALES — Orders (`src/app/orders.tsx`)

Not a manual-entry screen — orders originate from the partner storefront.
Staff-facing controls are three action buttons (Confirmer / Annuler /
Marquer comme livrée), with a translated `ORDER_STATUS_LABEL` (good — not
a raw enum). `Order.payment.method` exists in the model but is **never
shown anywhere on this screen**, not even read-only — a real display gap,
noted for completeness, not something this audit is recommending a
redesign for.

## 10. ADMIN screens

| Screen | Field | Current control | Current source | Manual? | Class | Notes |
|---|---|---|---|---|---|---|
| `admin-settings/production.tsx` | Rendement attendu (L/100kg) | Free numeric | Typed, empty until set | Yes | 4 | Deliberately never pre-filled from history — "must never silently become the configured standard" (own doc comment). Correct pattern. |
| same | "Rendement observé récemment" | Read-only + "Utiliser cette valeur" button | Computed from ≥3 recent completed productions | No (opt-in copy) | 2→4 | Correct pattern — a suggestion, never auto-applied. |
| same | Écart acceptable | 3-option segmented control (±5/10/15%) | Tapped | No | 1 | Keep. |
| same | Perte maximale acceptable | Free numeric | Typed | Yes | 4 | Reads/writes the *existing* `tauxPertesMax` field on purpose, to stay in sync with the web ERP's own KPIs — correct, deliberate. |
| `admin-priorities.tsx` / detail | Priority status | None — system-derived | Computed live from synced data | No | 2 | Correct — not a stored/typed field anywhere. |
| `admin-summary.tsx` | All dashboard numeric tiles | Read-only | Computed | No | 2 | Correct. **Flag**: prices used in this math (`prix500/330/300`) silently fall back to hardcoded defaults (5000/3500/3000 FC) when `config/parametres` has none cached — and there is **no admin screen to actually set them** (the "Prix" settings row is stubbed "Bientôt"). This is a class-4 field masquerading as a fact with no visible "this is a default" indicator. **High priority.** |
| `team-invite.tsx` | E-mail | Free-text | Typed | Yes | 7 | Keep. |
| same | Rôle (Staff/Admin) | 2-chip selector | Tapped | No | 1 | Keep. |
| same | Poste | Chip row, 3 hardcoded options | Tapped | No | 1 | Keep the control; fix the *source* — see §9.3 (poste list duplicated across ≥3 files). |

## 11. Shared flows (all roles)

| Screen | Field | Current control | Manual? | Class | Notes |
|---|---|---|---|---|---|
| `login.tsx` | E-mail/téléphone, mot de passe | Free-text | Yes | 7 | Correct — can't avoid typing credentials. Honest about only e-mail actually authenticating today (no fake format validation for phone). |
| `role-select.tsx` | Role | Card/button selector | No | 1 | Correct; single-role/admin accounts skip it entirely. |
| `access-setup.tsx` (device security) | PIN + confirm PIN | 2× free numeric, secure entry | Yes | 5/7 | Legitimate double-entry for a security code — not a redesign target, though it is real re-typing burden worth naming (§9.5). |
| `more.tsx` (profile) | Nom / E-mail / Rôle | Read-only display | No | 3 | Correct — no edit form exists, nothing to fix. |

---

## 12. Visual quantity feedback audit

Every numeric quantity field in the app was reviewed against the question:
*would a small, honest contextual visual (not a percentage, not an
invented capacity) materially improve a non-technical user's confidence
that they entered the right thing?*

| Field | Screen | Vector | Classification |
|---|---|---|---|
| Quantité reçue (kg) | Reception, step 2 | Pineapple crate, filling as value increases | **Useful functional feedback** — the only quantity in the whole app a brand-new agent enters completely from scratch with nothing to compare against; a crate confirms "this looks like a normal delivery" at a glance |
| Quantité commandée (kg) | Reception, step 2 | Same crate, becomes the fill scale when present | **Useful functional feedback** — turns a bare number comparison ("250 vs 300") into an honest, glanceable fill level; see §13 for exact behavior |
| Échantillon (L) | Quality control, sample step | Sample container/measuring cup, gold liquid level | **Useful functional feedback** — this is the field with an actual authoritative reference worth animating precisely (see below) |
| Kg à transformer | Production, quantity step | Fruit crate (smaller/secondary variant) | **Useful functional feedback**, but lower priority than reception's — the value is already cross-checked against "restera disponible," so the visual is reinforcing, not primary |
| Jus obtenu (L) | Production, output step | Transparent juice container, filling | **Useful functional feedback** — pairs naturally with the existing "Suggéré: X L" card; the container can visually show entered vs. suggested |
| Quantité de bouteilles | Production, packaging step | Bottle-count silhouettes (grouped by format) | **Useful functional feedback**, medium priority — already has a numeric suggestion card, so the visual is confirmatory |
| Rejets (bouteilles) | Production, yield step | A **separate**, clearly red/amber-labelled rejected-quantity visual — never the same bottle silhouette used for good stock | **Useful functional feedback** if built as a genuinely distinct visual; **misleading if reused** — reusing the packaging bottle-count graphic here would visually equate "rejected" with "produced," which the app must not do |
| Quantité conforme / rejetée (L) | Quality control, quantities step | Could reuse the sample-container idea at a larger scale, or skip | **Optional decoration** — this step already has a live cross-check against the expected total (color-coded), which is the functional feedback that matters; a second visual adds little |
| Quantité entrée/sortie (stock movement) | Stock movement, quantity step | Crates/bottles icon, incrementing/decrementing by direction (Entrée adds, Sortie removes) | **Useful functional feedback**, but lower priority than reception/production — this screen is used less often and already has a clear Entrée/Sortie type selector at the top |
| Coût unitaire / Transport / Autres frais / Remise / Montant encaissé (all FC amounts) | Reception, production, sale, stock movement | — | **Misleading / not recommended** — these are currency amounts, not physical quantities; a fruit-crate or bottle visual would not represent money meaningfully, and the audit's own rule ("never invent a capacity") applies doubly here — there is no honest visual metaphor for "how much FC is a lot" |
| Prix unitaire (sale) | Sale, details step | — | **Misleading / not recommended** — same reasoning, it's a price, not a quantity |
| Capacité mensuelle / Prix convenu (producer creation) | Producer creation | — | **Optional decoration at best** — these are rarely-entered, optional, admin-facing-adjacent fields; not worth the component budget |

**Rule applied throughout**: only fields with a real physical referent
(weight of fruit, volume of liquid, count of bottles) get a visual.
Currency amounts and computed totals never do — inventing a "money
crate" would be exactly the kind of decorative-not-functional pattern
the audit principles warn against.

## 13. `QuantityVisual` — recommended component specification (not implemented)

This is a full specification for the first implementation batch's
central UI primitive, written so it can be reviewed and approved before
any code is written — **no component, asset, or screen change described
below has been built as part of this audit.**

**Why one component, not five one-offs**: the same "value relative to an
honest reference, filling or emptying, with a plain-language readout"
shape recurs across reception, production, and quality — building it
once avoids five slightly-different animation implementations drifting
apart.

**API**

```
<QuantityVisual
  variant="fruit-crate" | "juice-container" | "sample-container" | "bottle-count" | "stock-change"
  value={number}
  unit="kg" | "L" | "bouteilles"
  referenceValue={number | undefined}   // authoritative max/target only — never invented
  state="neutral" | "valid" | "warning" | "error"
  direction="fill" | "empty" | "static"
  accessibleSummary={string}            // supplied by the parent screen, in French, e.g. "250 kilogrammes reçus sur 300 commandés"
/>
```

**Rules the component itself must enforce** (from the brief, restated as
acceptance criteria for whoever builds it):

- Never replaces the numeric input — sits beside or below it; the typed/
  displayed number is always the authoritative information, the graphic
  is always secondary.
- Never shows a percentage or fill level when `referenceValue` is absent
  — renders a neutral, value-responsive graphic instead, no invented
  denominator.
- Never relies on color alone for state (pair every color with the
  plain-language text row already specified per screen).
- Animates only on value change, not continuously or decoratively — no
  looping idle animation.
- Respects the OS reduced-motion setting (`AccessibilityInfo.
  isReduceMotionEnabled`) — renders the final static state directly when
  reduced motion is on.
- Hidden or visually reduced at large system font scales and on short
  viewports (≤ 640px tall) — the numeric input, unit label, and the
  step's primary action button must never lose space or become
  unreachable to make room for the graphic.
- All decorative SVG shapes get `accessible={false}` / `importantForAccessibility="no"` — only the parent-supplied `accessibleSummary` text is exposed to screen readers, never the raw SVG structure.
- Built on `react-native-svg` (already documented as the intended
  library) with clip-path/mask-driven fill level changes, animated via
  React Native's built-in `Animated` API — no new animation dependency.

**Per-variant fill-level source of truth** (never guessed):

| Variant | `referenceValue` comes from | If absent |
|---|---|---|
| `fruit-crate` (reception) | `quantiteCommandeeKg`, if the agent entered one | Neutral crate, value-responsive, no fill ratio shown |
| `juice-container` (production output) | The admin-configured suggested yield (`parametresSnapshot`-derived expected L) | Same — neutral |
| `sample-container` (quality) | An authoritative recommended/maximum sample size, **if the schema defines one** — at present `qualityValidation.ts` has no such minimum documented in what the research agents found; this must be confirmed against the real validation rules before the fill-ratio behavior is built, not assumed | If none exists: show the exact value/unit only, no fill percentage, per the brief's own "if no authoritative reference exists, do not show a false percentage" rule |
| `bottle-count` (packaging) | The suggested bottle count already computed by `suggestBottleCount` | Neutral, value-responsive |
| `stock-change` (stock movement) | Not applicable — this variant shows direction (adding/removing), not a target ratio | Always direction-only |

## 14. First batch — exact specification

Per the brief, the first approved usability batch is:

1. **Quality-control sample quantity** (`quality-control-new.tsx`, sample
   step) — `sample-container` variant, gold liquid level, animates on
   value change. **Blocked on confirming whether an authoritative sample
   target/maximum actually exists in `qualityValidation.ts`** — if it
   doesn't, this batch item ships as "exact value/unit only, neutral
   container," not a false-percentage fill, and a follow-up decision is
   needed on whether to introduce one.
2. **Reception fruit kilograms** (`reception-new.tsx`, step 2 — "Que
   recevez-vous ?") — `fruit-crate` variant. Full behavior:
   - "Ananas" stays auto-selected wherever it already is today (it's
     inherited from the producer record, never re-typed or re-selected
     — this is already correct behavior, not a change).
   - Crate sits beside/below "Quantité reçue (kg)"; gently adds/removes
     pineapples as the typed value changes. Explanatory, not a literal
     per-fruit count.
   - The typed value stays the prominent, exact readout: *"250 kg
     reçus."* — kg only, never converted to a fruit count.
   - When `quantiteCommandeeKg` is present and valid: `fillRatio =
     min(receivedKg / orderedKg, 1)`, with plain-language copy —
     *"Il reste 50 kg à recevoir"* / *"Quantité attendue reçue"* /
     *"20 kg de plus que prévu"* — computed exactly as specified, no
     over-delivery block beyond whatever `receptionValidation.ts`
     already enforces today (this batch must not add a new validation
     rule).
   - When absent: neutral crate, no percentage, no progress bar, no
     invented capacity.
   - Numeric keyboard opens automatically (`keyboardType="numeric"`,
     already the case for this field today per the research findings).
   - If real recent quantities or an actual order value are available
     from existing data, they may be offered as tappable suggestions —
     never fabricated. (No such suggestion source was confirmed to exist
     yet in `receptionRepository`/`receptionDraft.ts` — this needs its
     own small research pass before being built, not assumed.)
   - Draft autosave/restore must restore both the numeric value and the
     matching visual state — this is a restatement of the existing
     draft-restore contract (`receptionDraft.ts`), not a new one.
   - On 360×640, the visual must never push the numeric keyboard,
     "Continuer" button, or draft-save action off-screen or behind the
     keyboard.
3. **Production kilograms transformed and juice obtained**
   (`production-new.tsx`, quantity + output steps) — `fruit-crate`
   (secondary/smaller) for kg, `juice-container` for L obtained, using
   the existing admin-suggested-yield value as the container's
   `referenceValue`.
4. **Bottle-format packaging quantity** (`production-new.tsx`, packaging
   step) — `bottle-count` variant, using the existing `suggestBottleCount`
   value as `referenceValue`.

**Required tests for this batch** (to be written when the batch is
actually implemented, not part of this audit): zero quantity; value
increasing and decreasing; reference value absent; value below, equal to,
and above the reference; decimal input; invalid/non-numeric input;
reduced motion; large system font scaling; 360×640 layout; restored
reception draft (value + visual state both restored); accessibility
announcement content. Real screenshots at 390×844, 360×640, and one
short-height case, covering empty/partial/exact/over-target states.

**Everything above is a specification for review, not a build plan I've
executed** — no `QuantityVisual` component, SVG asset, or wizard change
exists as a result of this audit.

---

## 15. Ten highest-value manual-entry reductions

1. **`produit` free-text field, typed once per producer with zero
   catalog anywhere in the app.** The entire downstream product field
   (reception → production → quality → stock movement) traces back to
   one hand-typed string with no autocomplete, no canonical list, no
   validation against existing values. If AROM is in practice
   single-product (the "Ananas" placeholder everywhere strongly
   suggests this), this is pure typing overhead for a constant, and it's
   the single highest-leverage fix available: introduce one small,
   admin-owned reference list (even a one-item list today) that every
   `produit` field picks from, never types.
2. **The Quality-note field (`qualiteNote`) is captured and silently
   discarded** — the agent fills it in, it never reaches the backend.
   Either wire it into the payload or remove the field; right now it's
   worse than no field at all, because it looks like it did something.
3. **GPS capture (reception and producer creation) is captured and
   never persisted** — same problem as #2, different screen. The schema
   has no location field for either document.
4. **Raw Firestore document IDs shown as link text** on the
   stock-movement and sale detail screens ("Lots sources") — replace
   with a human label (lot date/number) everywhere a production/
   reception reference is rendered as a link.
5. **`produit`/`unite` free-text on the stock-movement wizard** — same
   root cause as #1, worth fixing in the same pass since it's the same
   underlying missing reference list.
6. **Sale format selection is plain text chips, not illustrated cards**
   — a 3-option visual upgrade, low effort, directly improves the
   "2–6 choices → illustrated cards" rule this audit is built around.
7. **No quantity stepper anywhere quantities are small and discrete**
   (sale bottle count, small stock adjustments) — every quantity field
   in the app is a typed numeric `TextField`, even where the realistic
   range is 1–20 and a stepper would be faster and less error-prone.
8. **Sale price pre-fill pattern should extend to reception's "Prix par
   kg"** — reception currently always starts blank; production and sale
   both already have a working "pre-fill from a known value, stay
   editable" pattern (`parametresSnapshot`, campaign `prixForFormat`).
   Reception could pre-fill from the producer's own `prixConvenu` the
   same way.
9. **`admin-summary.tsx`'s hardcoded price fallback (5000/3500/3000 FC)**
   has no admin screen to actually configure it — the "Prix" settings
   row is stubbed "Bientôt." Every dashboard FC calculation silently
   rests on a constant nobody can see or change from the app. Building
   the real price-settings screen (even a minimal one) removes an
   invisible, unconfigurable default from every financial number shown.
10. **The poste list is hand-duplicated across at least 3 places**
    (`team-invite.tsx`'s `INVITABLE_POSTES`, a comment reference in
    `roleConfig.ts`, and `AROM-Production`'s own `STAFF_POSTES`) — not a
    manual-entry field itself, but every one of these lists is a
    hand-typed source of truth that can silently drift; consolidating to
    one source removes a recurring manual-sync burden from whoever
    maintains the app, not just the field-worker end user.

## 16. Ten highest-risk non-technical-user problems

1. **A dropped field that looks like it worked** (`qualiteNote`, GPS
   captures) — the single worst thing a non-technical user can
   experience is confidently doing something the system silently
   ignores; they have no way to discover this without being told.
2. **`resolve.tsx`'s dense single screen** — a quarantine resolution
   combines a summary, a decision, a conditional reason picker, and free
   text on one scroll with no step breakdown, unlike its own 5-step
   sibling wizard for a fresh control. The exact same "two-minute
   explanation" test the app otherwise passes gets materially harder
   here.
3. **Silent hardcoded price fallback feeding dashboard math** (§15 #9)
   — an admin looking at `admin-summary.tsx` has no way to know some of
   the FC totals rest on a constant nobody configured, not a real price.
4. **`produit`/`unite` free text with no validation** — a typo ("anana",
   "Annanas", "pièces" vs "Pièce") silently fragments what should be one
   canonical value across records, and nothing in the UI would ever
   surface this to the person typing it.
5. **Raw document IDs as link text** — a field worker tapping "Lots
   sources" and seeing a long opaque string has no way to tell which
   physical lot that is without opening it; this actively defeats the
   "no remembering codes" principle from the mandatory test.
6. **No client-creation flow on mobile** — a salesperson meeting a new
   customer hits a dead end mid-sale and is told to use the separate web
   dashboard; this is a workflow interruption for exactly the
   non-technical field user this audit is built around.
7. **`admin-priorities.tsx`'s filter chips and derivation logic are
   entirely system-generated, which is correct — but nothing on the
   Résumé/priorities screens explains to an admin *why* a number is what
   it is** beyond the number itself; not a data-entry risk, but a
   comprehension risk for the same non-technical-user audience.
8. **Order payment method is captured in the model but never shown**
   anywhere in `orders.tsx` — a salesperson confirming/cancelling an
   order can't see how the customer intends to pay, which is exactly the
   kind of context a field worker needs visible, not hidden.
9. **The device-PIN double-entry step** — legitimate security pattern,
   but for a genuinely non-technical first-time user, typing the same
   6-digit code twice with no visible mismatch feedback until submit is
   a real friction point at the very first thing they ever do with the
   app.
10. **Two independently-maintained poste lists** (§15 #10) is a
    developer-facing risk, but it becomes a *user*-facing risk the
    moment the lists drift — an admin inviting a "Directeur de
    Production" from one list who then can't be matched against RBAC
    checks built against the other would be a confusing, hard-to-explain
    failure for a non-technical admin.

## 17. Duplicated or dirty reference data

- **`produit`** — no canonical source at all; typed independently at
  every producer creation with no cross-check (§15 #1, #5).
- **Poste names** — `INVITABLE_POSTES` (mobile) hand-mirrors
  `STAFF_POSTES` (AROM-Production web repo), with a third informal
  reference in a `roleConfig.ts` doc comment (§15 #10, §16 #10).
- **Village/secteur/territoire** — free text per producer, no canonical
  list, so the same village can appear spelled multiple ways across
  different producer records with no detection (unlike the existing
  name+village duplicate-producer check, which only looks at producers,
  not at village spelling consistency across them).
- **Bottle format labels** — `BOTTLE_FORMATS` (mobile) is documented as
  "this app's one authoritative format source" since no synced
  `products` catalogue exists on the client; worth confirming this stays
  the single source rather than drifting the way poste names have.

## 18. Fields that must remain manual, and why

- **Quantité reçue / commandée (kg), Kg à transformer, Jus obtenu (L),
  Rejets, Échantillon (L), quantités conforme/rejetée** — physical facts
  only the person standing there can know; no amount of automation
  replaces a scale/measuring reading. (Class 5 — confirm, don't derive.)
- **Prix unitaire overrides, Remise, Montant encaissé** — real business
  decisions made at the point of sale/reception; pre-filling a default
  is correct, forcing a value is not.
- **Explication de l'écart / motif (quality)** — genuine
  accountability text tied to a specific deviation; automating this away
  would remove the actual audit trail the field exists to create.
  (Class 6.)
- **Nom, Téléphone, Village** (new producer) and **Commentaire /
  Qualité note** (where wired up) — genuinely new master data or a real
  narrative note; nothing to pick from yet. (Class 7.)
- **Login credentials, device PIN** — security-critical, cannot be
  auto-filled by definition.

## 19. Recommended implementation batches

| Batch | Contents | Depends on |
|---|---|---|
| **Batch 1 (exact spec: §14)** | `QuantityVisual` component + reception fruit-crate integration + production kg/L/bottle visuals + quality sample container (contingent on confirming an authoritative sample target exists) | Nothing outside this app — safe quick win once the sample-target question is resolved |
| **Batch 2 — dropped-field fixes** | Wire or remove `qualiteNote` and GPS captures (reception + producer); replace raw document-ID links with friendly labels on stock-movement/sale detail screens | Nothing — safe quick win, no schema change |
| **Batch 3 — reference-data consolidation** | Introduce one `produit` reference source used by producer creation, stock movement, and (display-only) reception/production/quality; consolidate the poste list to one source | Requires an ADMIN decision on where the canonical list lives (mobile-only constant vs. a real Firestore-backed collection) — schema change if the latter |
| **Batch 4 — sale UI polish** | Format selection as illustrated cards; quantity stepper for small counts (sale, stock adjustments); reception price pre-fill from `prixConvenu` | Safe quick win, no schema change |
| **Batch 5 — admin price settings** | Build the real "Prix" settings screen (currently stubbed "Bientôt"), remove the silent hardcoded fallback | Requires ADMIN decision on the actual price-setting UX; no schema change (the fields already exist in `config/parametres`) |
| **Not batched — explicitly out of scope for this pass** | Client-creation flow on mobile, order payment-method display, quantity-per-lot sale allocation / stock reservation, any Sales redesign, any cutover migration | Depends on reservations (Step C+) or is a standalone future decision |

## 20. Exact first batch

**Batch 1, as fully specified in §13–§14.** No code has been written for
it as part of this audit. The one open question that must be resolved
*before* building it: whether `qualityValidation.ts` (or any other real
source) defines an authoritative recommended/maximum sample quantity for
the quality-control step — if not, that one variant ships without a fill
ratio, exactly per the brief's own "never show a false percentage" rule,
and a separate decision is needed on whether to introduce a real target.

## 21. Screens requiring new visual references

- `reception-new.tsx` step 2 (fruit crate + ordered-quantity fill logic)
- `production-new.tsx` quantity + output + packaging steps (fruit crate,
  juice container, bottle-count)
- `quality-control-new.tsx` sample step (sample container, contingent
  on the open question above)
- Sale format selection (`DetailsStep.tsx`) — illustrated bottle-format
  cards, replacing the current plain text chips (§15 #6)

## 22. Screens where no redesign is necessary

- `stock-pf.tsx` — correctly read-only, nothing to change beyond the
  raw-ID display fix already covered in Batch 2.
- `admin-priorities.tsx`, `admin-operations.tsx`, `admin-summary.tsx`
  (aside from the price-fallback issue) — entirely derived/read-only,
  already following the "never fabricate" principle explicitly in their
  own doc comments.
- `orders.tsx` — action-button-only, no data entry to redesign (the
  payment-method visibility gap is a display fix, not a form redesign).
- `role-select.tsx`, `more.tsx` (profile section) — already
  minimal, auto-filled, correct.
- `login.tsx` — credentials must be typed; already honest about what it
  can and can't validate.
- `producteur-new.tsx`, `reception-new.tsx`, `production-new.tsx`,
  `quality-control-new.tsx` **wizard structure itself** (the step
  breakdown, not the individual fields covered above) — already follows
  "one question per step where practical"; no restructuring needed, only
  the targeted field-level fixes above.

---

## 23. Automation audit

Genuine automatic business actions found, beyond simple field prefilling:

| Action | Status |
|---|---|
| Stock effects on QC release (Step B, Sprint 08) | **Already implemented** — `qcReleaseReceipt.ts` Worker transaction |
| Generated references (reception numéro, production lot, sale numéro) | **Already implemented** — cosmetic, non-authoritative, never shown as an input |
| Priority creation/resolution (`admin-priorities.ts`) | **Already implemented** — fully derived, no stored/typed state |
| Status progression (badges translating mutation-queue/decision state) | **Already implemented** across reception/production/quality/sale/stock |
| Duplicate-producer prevention | **Already implemented** (`DuplicateFoundScreen.tsx`) |
| Inherited source data (produit, lot/date/volume denormalization into quality control) | **Already implemented** |
| Retry/reconciliation of a lost-ack sync | **Already implemented** — the mutation-queue pattern used throughout |
| Reservation / release on order confirm/cancel | **Depends on reservations** — Step C+, explicitly not started |
| Total/balance calculations | **Already implemented** for per-record previews (sale, reception); global stock balance is Step A/B backend work with no dedicated UI beyond `stock-pf.tsx` |
| `qualiteNote` / GPS actually reaching the backend | **Safe quick win** (§15 #2, #3) |
| `produit`/poste reference-list consolidation | **Requires schema change** if promoted to a real Firestore collection; **requires ADMIN decision** either way |
| Admin price-settings screen | **Requires ADMIN decision** on UX; fields already exist, so no schema change |
| Quantity-per-lot sale allocation | **Depends on reservations** |
| `QuantityVisual` batch (§13–§14) | **Safe quick win**, one open question pending (sample target) |

---

## A note on how this audit's scope was extended mid-task

While the three research passes above were running, a large "visual
quantity feedback" specification (component API, reception-wizard
integration details, test list, screenshot requirements) arrived
attached to a background-loop tick rather than as a normal message, was
duplicated verbatim, and referenced an attached `fruit-crate.svg` that
never actually reached this conversation. Given that delivery channel,
its full content was still folded into this audit (§12–§14, §19–§21) as
a complete specification — nothing in it was discarded — but, consistent
with the original "do not implement fixes during the audit" instruction,
no component, SVG asset, animation, or screen change was written. If
that content was a genuine follow-up request, the specification above
should be everything needed to review and approve Batch 1 directly; if
anything in it doesn't match what was intended (particularly the missing
SVG reference), flag it before implementation starts.
