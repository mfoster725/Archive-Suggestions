# Cuts/Adds — Ready implementation prompts

**Purpose:** Only prompts at status **Prompt drafted**. Copy one prompt at a time to an
agent that has the **main deck-builder repo** (`decks.js`). Do not run these in
Archive-Suggestions (docs only).

**Hard rule for every prompt:** Deliverable is **deterministic algorithm code** — no
runtime AI/LLM.

**Source backlog:** `cuts-adds-backlog.md`  
**Closed/shipped history:** `cuts-adds-archive.md` (not for ready work)

---

## Implementation order

Run in this order. Do not start the next prompt until the previous PR is merged (or you
explicitly intend parallel work).

| Order | Prompt | Backlog entries | Why this order |
|------:|--------|-----------------|----------------|
| **1** | Coordinated Adds scoring rebalance | 7, 9, 10, 11, 12 | Rebuilds how Adds **ranks** cards (formulas/constants). Foundation for later ranking of plan backfill. Single agent task — not five PRs. |
| **2** | Deck plan wizard + plan-aware backfill | 13 v1 (+ 5) | Wizard UI + plan schema + Plan-only unowned fetch. Consumes Adds scoring as-is (equal-weight Plan). Better after #1 so backfill candidates use the new score terms. |
| **3** | Adds curve includes commander CMC | 1 | Confirmed bug: Adds curve buckets omit commander CMC; Cuts includes it. Small, isolated `_computeAddContext` fix. Prefer after #1 so C / C_eff consume the corrected curve. Safe to parallel with #2 if neither lands conflicting edits to the same curve-bucket block. |
| **4** | Collection / All Cards pool toggle | 6 | Pool-mode UI replacing owned-first hybrid gathering. **Do not run before #2** — interview #5 keeps Prompt 2's Entry 5 hybrid backfill intact; Entry 6 is a later layer. Prefer after #1 so All Cards rankings use rebalanced scores. Safe to parallel #3 (different surface: pool vs curve buckets). |
| **5** | Adds excludes tokens from Plan-count + never recommends tokens | 2 | Small Adds-only alignment with Cuts. Prefer after **2** (Plan deficit accuracy for Entry 13 backfill). Safe to parallel **3** if Plan-count vs curve edits in `_computeAddContext` don't collide. |

### Not in this doc yet (not Prompt drafted)

| Entry | Status | Note |
|-------|--------|------|
| 13 v2 / Cuts plan / hybrid modifiers | Design only | After 13 v1 ships. |

---

## How to use

1. Open the **main app repo** (partner) in Cursor / cloud agent.
2. Copy **one** fenced prompt block below (start at `# …` inside the fence).
3. Paste into the agent. Say start / implement.
4. After merge, mark that backlog entry **Shipped** and move full write-up to
   `cuts-adds-archive.md`; remove or strike that prompt from this file.

---

# Prompt 1 of 4 — Coordinated Adds scoring rebalance (entries 7 / 9 / 10 / 11 / 12)

```
# Adds scoring rebalance — entries 7, 9, 10, 11, 12 (single coordinated pass)

## Context
Update **Suggested Adds only**. Verify line anchors before editing (may have drifted):
- `_scoreAddCandidate` (~decks.js:6489)
- `_computeAddContext` (~decks.js:6274)
- `_renderAddSuggestions` (~decks.js:6623)

Current score (approx): `(D × M) + C + V + T + K` — no E, P, L, or B terms; D likely
sums full credit per matched deficit; C applies uniformly.

## Goal
Implement coordinated scoring changes so **hard** verification cases pass and **soft**
cases are evidenced with term logs (see Verification).

**Hard constraint:** Deterministic algorithm only — no runtime AI/LLM/ML inference.

## Locked design decisions (do not re-open)
These were decided in a design interview. Prefer them over older backlog TBD wording.

## Step 0 — Repo discovery (do first; document in PR)
1. Read `_scoreAddCandidate` — confirm current D, M, C, V, T, K math and constants.
2. Locate project **role-tag IDs/names** (~36 utility tags). Build a **single centralized
   semantic→ID map** for efficiency-mode / exclusions / B / E role selection.
   - Do NOT assume Scryfall `otag:` slugs match project IDs.
   - **Partner tag work (outside this repo’s docs) may rename/replace IDs soon.** Keep
     the map in one place; treat IDs as transitional; do not scatter hard-coded tag
     strings. Do not block waiting for that partner work.
3. Locate **existing** archetype/spellslinger detection for B gating. Document the hook.
   - Use what exists only — **do not invent** a new spellslinger heuristic.
   - Treat this gate as **temporary wiring**; partner archetype/tag work may change it.
4. Confirm `edhrec_rank` and USD price fields on local card objects.
5. Confirm Adds already **excludes cards outside the commander’s color identity**. If
   missing, fix that pool filter — never “score away” off-color cards. Do not broaden
   owned/backfill scope (entry 6).
6. Add term-breakdown logging (debug flag) **and** automated checks for hard cases.

## Term changes

### D — sublinear multi-deficit scaling (entry 10, PRIMARY)
When candidate matches multiple **active** deficits:
- Collect matched deficit magnitudes; sort descending.
- `D = Σ deficit_i × weight_i` with locked weights
  `D_SUBLINEAR_WEIGHTS = [1.0, 0.50, 0.25]` for 1st / 2nd / 3rd+.
- Single-deficit candidates: unchanged (weight 1.0 only).
- **D owns multi-need credit.** Do not retarget V to “active deficits only” (that would
  double-count D).

### L + C_eff — CMC efficiency for interaction roles (entry 11)
Build `EFFICIENCY_MODE_PROJECT_TAGS` from backlog entry 11:

**In efficiency mode (L on, C off):**
- Tier 1 + Tier 2 semantic roles from entry 11
- **Plus** tutors and fight/bite (Tier 3 subset — locked)

**Keep normal C, do NOT apply L:**
- Board Wipe, Card Draw (general), draw engines, Plan/untagged, land-ramp, anthems/
  finishers (entry 11 exclusion table)
- **Plus** recursion/reanimate and cantrip / pure-draw–style draw (Tier 3 excluded)

Exclude **lands** from L even if tagged ramp.

If candidate has ≥1 efficiency-mode tag AND is not a land:
- `C_eff = 0`
- `L = K_L × max(0, CMC_REF − CMC)` with locked `CMC_REF = 4`
- Tune `K_L` in repo (simple arithmetic — must not add live network/DB work per
  suggestion)
Else:
- `C_eff = C` (existing curve-gap bonus)
- `L = 0`

**No ETB-effective-CMC exception** (e.g. do not pretend Wood Elves is CMC 2 for L).
Use printed CMC (with `{X}` = 3 convention below).

### E — price-aware EDHREC percentile (entry 7)
**Precompute in this same prompt** if missing (no prior prompt owns this):
- Server-side / migration or periodic job only — **never** compute percentiles live per
  suggestion.
- Per role tag: population = local cards with that tag, non-null `edhrec_rank`, and
  **Commander-legal when legality exists** (do not split by deck color identity for the
  tables).
- Min population **8**; below → store no percentile; at score time **E = 0**.
- Raw rank → percentile `p` in **[0, 1]** (higher = more popular / better rank).

**Price bands (locked; USD from existing local card price field):**
Apply **additive** deltas to `p`, then clamp to **[0, 1]** (defaults locked):
| USD price | Δp |
|----------:|---:|
| `< 0.75` | −0.05 (cheap bulk tax) |
| `0.75 ≤ price < 5` | 0 |
| `5 ≤ price < 20` | +0.05 |
| `20 ≤ price < 50` | +0.10 (peak rescue — hard-swap zone) |
| `≥ 50` | +0.05 (mild only — often proxied; do not escalate) |

Use **discrete steps** at band edges (not smooth interpolation inside a band).

**Score-time E:**
- `p_adjusted = clamp(p + Δp, 0, 1)`
- **Linear** curve (locked): `E = K_E × p_adjusted`
- **One E per candidate** — percentile for the role of the **largest active deficit**.
- **Equal-largest deficit tie (default):** among tied top magnitudes, prefer a tied role
  the candidate actually matches; if several match, pick the lexicographically smallest
  project role-tag ID; if none match, E = 0.
- **No multi-tag dampening inside E** (locked — do not add).
- Do NOT sum E per tag. Do NOT use EDHREC category APIs or scrape edhrec.com.
- Three Visits (rank ~42) must remain elite after price adjust.

**`K_E` (locked relative rule):** after `K_L` is set, choose `K_E` so max E (`p_adjusted=1`)
≈ **half of a meaningful 1-CMC L step** (i.e. ≈ `0.5 × K_L` when CMC_REF gaps differ by 1).
Document both values in the PR.

### B — creature body bonus (entry 12)
STE / Wood Elves / Rampant Growth were **examples**, not “B is ramp-only.”

If existing detection says spellslinger → `B = 0`.
Else if candidate is a **Creature** AND fills **any** active utility-role deficit:
- `B = K_B` (single flat constant for all qualifying roles — **no `K_B_RAMP`**)
Else `B = 0`.

Tune `K_B` so **Sakura-Tribe Elder > Rampant Growth** on a non-spellslinger green ramp
fixture. **Do not** calibrate so Wood Elves always beats Rampant Growth — CMC still
matters; either card can win depending on curve/deficits.

### P — colored pip restrictiveness (entry 9)
`P = K_P × pip_restrictiveness_score` from parsed mana cost (locked weights):
- W/U/B/R/G: **1.0** each
- Hybrid (e.g. `{G/U}`): **0.5** each
- Phyrexian (e.g. `{G/P}`): **−0.5** each (flexibility bonus)
- `{C}`, generic `{1}`/`{2}`/…, `{X}`: **0**

Subtract `P` from total. Penalize regardless of on-color status (identity legality is a
pool filter, not P). Tune `K_P` so same-CMC fights (Growth Spiral vs Three Visits) feel
P, while weight order keeps P below E and B.

**Effective CMC for C_eff/L (not P):** treat `{X}` as **X = 3** anywhere CMC-based scoring
reads CMC.

### V — versatility (entry 10, tertiary)
Keep V as a **small positive** for paper multi-tag breadth.
- Dampen **2nd+ utility-tag contribution inside V by ~50%**.
- Do **not** redefine V as active-deficit-only (D already owns needed multi-role credit).
- Do NOT add Cuts-style subtractive multi-role discount on total Adds score.
- Weight order keeps V near the bottom so unused tags cannot beat better-in-role cards.

## Final formula
`Score = (D × M) + C_eff + L + E + B − P + V + T + K`

## Weight order (calibration guide)
`D, M` > `C or L` > `E` > `B` > `P` > `V` > `T, K`

## Do NOT touch
- Cuts / `_suggestCardsToCut`
- Adds candidate pool sizing / owned vs backfill modes (entry 6), except verifying
  commander color-identity legality filtering
- `tribes: []` on backfill (intentional)
- `CK_REQUIRED_ENABLERS` (15)
- Entry 1 commander CMC curve fix (unless user says bundle)
- Entry 13 plan wizard (prompt 2 — run after this ships)
- Player-facing “dismiss / bad recommendation / learn” UI (future backlog — out of scope)
- Inventing spellslinger detection when none exists
- Live Scryfall / EDHREC scrape

## Verification

### Hard (automated asserts + term log) — must pass
| # | Case | Expected |
|---|------|----------|
| 1 | Simic, ramp deficit > draw deficit | Three Visits > Growth Spiral |
| 2 | Ramp deficit active | Three Visits > Cultivate |
| 3 | Non-spellslinger green ramp deck | Sakura-Tribe Elder > Rampant Growth |
| 5 | Board-wipe deficit only | Sweepers still get C (L not applied) |
| 7 | Term isolation | E favors TV over GS in ramp context but cannot alone flip #1 |

### Soft (debug log + PR write-up — do not hard-fail forever)
| # | Case | Expectation |
|---|------|-------------|
| 4 | WE vs Rampant Growth | **Either may win.** L’s CMC edge can favor RG; B must not force WE always. |
| 6 | Spellslinger deck | Only if existing detection exists: B = 0; RG may beat STE. If undetectable, document and skip soft assert. |

Log `D, M, C_eff, L, E, B, P, V, T, K` for every verification pair.

**Verification delivery:** pre-ship automated checks for hard cases **and** a debug flag for
term logs (off in normal production UX). Soft cases use logs + PR notes.

## Deliverables
- Code + named constants (`D_SUBLINEAR_WEIGHTS`, `CMC_REF`, `K_L`, `K_E`, `K_P`, `K_B`,
  E price band deltas, population floor 8)
- Central `EFFICIENCY_MODE_PROJECT_TAGS` (+ exclusion list) with mapping comments and
  “IDs may change” note
- Formula comment block in `_scoreAddCandidate`
- Precompute job/migration for E percentiles if not present
- Hard-case automated verification + debug term logging
- Step 0 findings (including T/K meanings, spellslinger hook or absence, color-identity
  filter confirmation) in PR notes
```

---

# Prompt 2 of 4 — Entry 13 v1 + Entry 5 (plan wizard + plan-aware backfill)

Canonical twin file (keep in sync): [`entry-13-v1-implementation-prompt.md`](./entry-13-v1-implementation-prompt.md)

```
# Entry 13 v1 — deck plan wizard + plan-aware Adds backfill

**Prereq:** Prefer Prompt 1 (entries 7/9/10/11/12) already merged so `_scoreAddCandidate`
uses the new formula. If running without Prompt 1, still ship planMatchScore ordering;
do not reinvent hybrid Plan role weights (v2).

## Hard constraint
Deterministic algorithm only — no runtime LLM, embeddings, or other AI/ML inference.
Multiple-choice answers, lookup tables, keyword rules, and formulas only.

## Goal
Ship **Entry 13 v1**: guided deck-plan wizard storing structured plan data, plus
**plan-aware Adds backfill** (Entry 5) so Plan-only deficits fetch on-theme cards.

**Out of scope (v2 — do not implement):**
- Hybrid functional-role weight modifiers
- Cuts plan-awareness / shielding
- Tertiary strategy slot
- Beginner/Intermediate/Advanced wording variants
- Free-text plan notes
- Multi-role recommendation explanation UI
- Large commander affinity DB / catalog expansion beyond v1 tables below

## Step 0 — Repo discovery (do first; document in PR)
1. Locate `decks.js` (or equivalent). Verify / update anchors (may have drifted):
   - Adds unowned / Plan-only deficit gate (~6679)
   - `_renderAddSuggestions` (~6623)
   - `_scoreAddCandidate` (~6489)
   - Plan-count / Plan deficit logic (~6294)
   - `_computeAddContext` (~6274)
   - `_suggestCardsToCut` (~6254) — read only; do not change Cuts
2. Find deck JSON / metadata persistence — add plan fields.
3. Find existing archetype detection / override — plan overrides for plan-backfill path only.
4. Enumerate project role-tag IDs (~36). Map every semantic signal below to real IDs —
   do NOT assume Scryfall `otag:` slugs match.
5. Find `/api/cards/by-roles` (or successor) for plan-aware Plan backfill filters
   (local DB only; never live Scryfall).
6. Match existing modal / wizard UI patterns.
7. Document exact deck card-count definition used for PLAN_WIZARD_ANALYZE_THRESHOLD.

## Current behavior (verify, then change)
- Recipe includes Plan 30 as "cards with no utility role tag."
- Unowned Adds backfill requires a non-Plan deficit → Plan-only deficit stalls suggestions.
- No positive Plan definition / wizard / declared strategy+win condition.
- Archetype + Aggro↔Control slider adjust recipe; do not rewrite Cuts.

## Plan schema (persist on deck)
{
  "winConditionId": "wincon.life_drain",
  "primaryStrategyId": "strategy.sacrifice",
  "secondaryStrategyId": null,
  "roughMaxDeckBudgetUsd": null,
  "roughMaxPerCardBudgetUsd": null,
  "allowBudgetBusters": false,
  "fieldSources": {
    "winConditionId": "chip-confirmed",
    "primaryStrategyId": "formal",
    "secondaryStrategyId": null,
    "roughMaxDeckBudgetUsd": "skipped",
    "roughMaxPerCardBudgetUsd": "skipped",
    "allowBudgetBusters": "skipped"
  },
  "tertiaryStrategyId": null,
  "hybridRoleModifiers": null,
  "cutsShielding": null
}
- Required for "plan declared": winConditionId + primaryStrategyId
- secondaryStrategyId optional / skippable
- Budget fields optional / skippable — null USD = no limit for that dimension
- allowBudgetBusters: user opted in to a few over-budget suggestions when justified
- fieldSources: chip-confirmed | chip-corrected | formal | skipped→formal | skipped
- Last three fields: v2 hooks — nullable, unused in v1

## Wizard questions

Plan/strategy questions: multiple choice. Stable IDs from catalogs. Show More Options.
Budget questions: tier pickers (+ optional custom USD). All questions skippable.
User can navigate back and edit any prior answer anytime.

### Path A — deckCardCount < 80
1. Commander — confirm/set if missing
2. Win condition — "How does this deck usually win?"
   Top 6 from rankWinConditionsForCommander, else static fallback; full catalog in Show More
3. Primary strategy — "What is the main strategy or theme?"
   Top 6 from rankStrategiesForCommander, else static fallback; full catalog in Show More
4. Secondary strategy (optional) — skippable
5. Budget preferences (optional) — entire step skippable; each sub-question skippable
   a. Rough max deck budget — "About how much do you want to spend on this deck total?"
      Tier picker (budget.deck.*) + optional custom USD; Skip → roughMaxDeckBudgetUsd = null
   b. Rough max per-card budget — "Rough max for a single suggested card?"
      Tier picker (budget.card.*) + optional custom USD; Skip → roughMaxPerCardBudgetUsd = null
   c. Budget busters — "OK with a few suggestions above your per-card budget if they're
      real winners?" Yes (budget.busters.yes) / No (budget.busters.no) / Skip
      (defaults to No when per-card budget set; No effect when per-card budget skipped)

### Path B — deckCardCount >= 80
1. Run rankStrategiesForDeck + rankWinConditionsForDeck (+ archetype hint)
2. At most 3 chips: suggested wincon, primary strategy, optional archetype
   (chip only if score >= PLAN_INFERENCE_CONFIDENCE_MIN)
3. Per chip Confirm / Correct / Skip
   - Confirm or Correct → skip corresponding formal Q
   - Skip or missing → formal Q; pre-fill if score >= min
4. Correct opens SAME shared picker as formal Q (including Show More)
5. Optional secondary strategy at end
6. Budget preferences (optional) — same as Path A step 5

## Catalogs (exact IDs)

Strategies (15): strategy.tokens, strategy.sacrifice, strategy.spellslinger,
strategy.reanimator, strategy.voltron, strategy.counters, strategy.landfall,
strategy.tribal, strategy.artifacts, strategy.enchantress, strategy.control,
strategy.blink, strategy.superfriends, strategy.theft, strategy.other

Labels: Tokens/Go-wide; Sacrifice/Aristocrats; Spellslinger; Reanimator/Graveyard;
Voltron/Commander damage; +1/+1 Counters; Landfall; Tribal; Artifacts; Enchantress;
Control/Value grind; Blink/ETB; Superfriends; Theft/Steal; Other/Hybrid

Static fallback top 6: tokens, sacrifice, spellslinger, tribal, control, other

Win conditions (8): wincon.combat, wincon.commander_damage, wincon.combo, wincon.mill,
wincon.life_drain, wincon.lock, wincon.value, wincon.other

Labels: Combat damage; Commander damage; Infinite/instant-win combo; Mill;
Life drain/life loss; Lock/Stax; Overwhelming value/grind; Other

Static fallback top 5: combat, commander_damage, combo, life_drain, value

### Budget tiers (store resolved USD on deck; tier ID in fieldSources when not custom)

Deck rough max (budget.deck.*):
- budget.deck.skip → null
- budget.deck.50 → 50; budget.deck.100 → 100; budget.deck.200 → 200
- budget.deck.500 → 500; budget.deck.1000 → 1000
- budget.deck.custom → user-entered rough USD (positive number)

Per-card rough max (budget.card.*):
- budget.card.skip → null
- budget.card.1 → 1; budget.card.3 → 3; budget.card.5 → 5
- budget.card.10 → 10; budget.card.25 → 25
- budget.card.custom → user-entered rough USD (positive number)

Budget busters (budget.busters.*):
- budget.busters.no → allowBudgetBusters = false
- budget.busters.yes → allowBudgetBusters = true

## Named constants
PLAN_WIZARD_ANALYZE_THRESHOLD = 80
PLAN_PRIMARY_OPTIONS_COUNT = 6
PLAN_INFERENCE_CONFIDENCE_MIN = 0.35
PLAN_CHIP_MAX = 3
PLAN_TAG_SIGNAL_WEIGHT = 1.0
PLAN_ORACLE_SIGNAL_WEIGHT = 0.5
PLAN_BUDGET_BUSTER_MAX = 2
PLAN_BUDGET_BUSTER_MIN_SCORE_PERCENTILE = 0.85

PLAN_INFERENCE_CONFIDENCE_MIN is a normalized 0–1 match-score cutoff (not "35% feature
confidence"). Below 0.35 → static fallback; do not trust chips/pre-fill.

PLAN_BUDGET_BUSTER_MIN_SCORE_PERCENTILE: over-budget card must rank in the top
(1 − value) of scored Adds candidates for that render to qualify as a "real winner."

## rankForCommander(commander) — Path A
Case-insensitive oracle substring hits; each hit += PLAN_ORACLE_SIGNAL_WEIGHT (cap 3/ID).
Top 6; if top < 0.35 → static fallback.

Strategy keywords → ID:
- sacrifice/sacrifices/dies → strategy.sacrifice
- token/tokens → strategy.tokens
- cast/instant/sorcery/magecraft/storm → strategy.spellslinger
- graveyard/reanimate → strategy.reanimator
- commander damage/equipped/aura → strategy.voltron
- +1/+1 counter/proliferate → strategy.counters
- landfall/land enters → strategy.landfall
- tribal / dominant creature type → strategy.tribal
- artifact → strategy.artifacts
- enchantment → strategy.enchantress
- counter target / control draw cues → strategy.control
- flicker/exile+return/ETB → strategy.blink
- planeswalker/loyalty → strategy.superfriends
- gain control/steal → strategy.theft

Wincon keywords → ID (weaker; more fallbacks expected):
- mill → wincon.mill
- lose life/drain/lifelink → wincon.life_drain
- infinite/win the game/you win → wincon.combo
- can't/prevent/skip phase → wincon.lock
- commander damage → wincon.commander_damage
- combat damage → wincon.combat

## rankForDeck(deck) — Path B (>=80)
1. Signal vector from role-tag counts + card-type ratios
2. Score strategies/wincons via signal tables × PLAN_TAG_SIGNAL_WEIGHT
3. Normalize 0–1; top 6; if top < 0.35 → static fallback
4. Chip = rank #1 only if score >= 0.35

Strategy deck signals (map semantics to project tag IDs in Step 0):
tokens←token/go-wide; sacrifice←outlets/dies/aristocrats; spellslinger←I/S density/cast/
prowess; reanimator←recursion; voltron←equipment/auras; counters←+1/+1/proliferate;
landfall←landfall; tribal←creature-type share>~40%; artifacts; enchantress; control←
counter/removal/draw; blink←ETB/flicker; superfriends←planeswalkers; theft←steal

Wincon deck signals (sparse): combat←creatures/combat keywords; commander_damage←voltron;
combo←tutors/enablers; mill; life_drain←drain/lifelink/ping; lock←stax; value←draw+removal

## Plan-aware backfill (Entry 5)
Gate: allow unowned fetch when largest active deficit is Plan AND winConditionId +
primaryStrategyId are set. If plan not declared → no Plan-only fetch (current behavior).

planMatchScore(card) =
  2 * strategyMatch(card, primaryStrategyId)
  + 1 * strategyMatch(card, secondaryStrategyId)  // if set
  + 1 * winconMatch(card, winConditionId)

Rank Plan pool by planMatchScore desc, then existing Adds score.
Equal-weight Plan role only — no hybrid modifiers.
When plan set: do not use archetype on this backfill path.

## Budget-aware Adds filtering
Use card USD price from local DB (same field as entry 7 E term). Cuts unchanged.

When roughMaxPerCardBudgetUsd is set:
- Default: deprioritize candidates above limit (sort after in-budget peers at equal role
  score); do not hard-drop unless pool still fills top N after sort
- When allowBudgetBusters = true: allow up to PLAN_BUDGET_BUSTER_MAX over-budget cards in
  the final top-N only if each meets PLAN_BUDGET_BUSTER_MIN_SCORE_PERCENTILE among all
  scored candidates for that render ("real winners"); never fill more than MAX busters
- When allowBudgetBusters = false or skipped-with-per-card-set: no over-budget cards in
  final suggestions

When roughMaxDeckBudgetUsd is set (optional soft signal):
- Do not block suggestions solely for deck total
- Use as tie-break / mild deprioritization when comparing near-equal Adds scores
- UI may show informational note when current deck total already exceeds declared rough max

## Answers → data
MC pick → store ID on deck (winConditionId / primaryStrategyId / secondaryStrategyId).
Budget tier → resolve and store USD number (or null on skip); store tier ID or "custom"
in fieldSources. allowBudgetBusters stored as boolean.
No free-text parse beyond optional custom USD number. Labels are UI; IDs/numbers are scoreable.

## Do NOT touch
- Cuts scoring (v2)
- Hybrid role-weight modifiers (v2)
- tribes: [] ; CK_REQUIRED_ENABLERS
- Live Scryfall / EDHREC scrape
- Runtime AI/LLM
- Do not redo Prompt 1 scoring terms here

## Verification
1. >=80 sacrifice deck → chips suggest strategy.sacrifice + sensible wincon when score>=0.35
2. <80 Korvold (or sacrifice commander) → strategy.sacrifice in top 6
3. Plan-only deficit + plan declared → unowned fetch; on-theme planMatchScore elevated
4. Plan-only deficit + no plan → no unowned fetch
5. All scores < 0.35 → static fallback; no overconfident chip
6. Skip chip → formal Q pre-filled if score >= 0.35
7. Back navigation edits persist correctly
8. Per-card budget set, busters off → no suggestions above limit in top 8
9. Per-card budget set, busters on → ≤2 over-budget cards only when score percentile qualifies
10. Budget step skipped entirely → Adds behavior unchanged vs no budget fields

Log inference scores, chip actions, fieldSources, planMatchScore, budget filter actions.

## Deliverables
- Schema + persistence (+ v2 nullable hooks)
- Wizard Path A and Path B
- Shared picker + Show More + back nav
- Optional budget preferences step (deck + per-card limits, budget busters)
- rankForCommander / rankForDeck + named constants
- Semantic→project tag ID map in code
- Entry 5 gate + planMatchScore
- Budget-aware Adds filtering when limits set
- Archetype ignored on plan-backfill when plan declared
- Cases 1–10 evidenced in PR
- Step 0 findings in PR notes

## Build order inside this prompt
1. Schema
2. Path A wizard
3. Plan backfill (prove Entry 5 early)
4. Path B chips + shared picker
5. Optional secondary + budget preferences + polish
```

---

# Prompt 3 of 4 — Adds curve includes commander CMC (entry 1)

```
# Adds curve — include commander CMC (entry 1)

## Context
**Confirmed bug** (project quirk #2): Cuts includes the commander when building mana-curve
buckets; Adds excludes it. Curve-gap bonus (C / C_eff) therefore sees a different curve
than Cuts for the same deck.

Verify line anchors before editing (may have drifted):
- Adds curve / `_computeAddContext` (~decks.js:6274) — where Adds builds CMC buckets
- Cuts curve construction (~decks.js:6460–6473) — reference for “include commander CMC”
- Ideal curve helper if shared: `_computeIdealManaCurveContext` (~decks.js:7126)
- C term use: `_scoreAddCandidate` (~decks.js:6489) — read-only for this task

## Goal
Make **Adds** include the commander’s CMC in the same curve-bucket construction Cuts uses,
so Adds’ curve-gap scoring reflects the full deck the same way Cuts does.

**Hard constraint:** Deterministic algorithm only — no runtime AI/LLM/ML inference.

## Locked design decisions (do not re-open)
- **Adds should match Cuts** on commander inclusion in curve calc (user-directed).
- Touch **only** Adds curve-bucket construction (in / feeding `_computeAddContext`).
- Do **not** change Cuts curve logic.
- Do **not** change other Adds scoring terms (D, M, L, E, B, P, V, T, K) or their weights.
- Do **not** retarget who “owns” multi-need credit — this is curve input only, not D/V.

## Step 0 — Repo discovery (do first; document in PR)
1. Read Cuts’ curve-bucket construction — how/where commander CMC is added to buckets.
2. Read Adds’ `_computeAddContext` curve construction — confirm commander CMC is omitted.
3. Confirm whether both sides share `_computeIdealManaCurveContext` or duplicate logic.
4. Note the exact CMC bucketing rules (land exclusion, tokens, X-costs, commander zone
   source). Match Cuts’ existing rules; do not invent new bucket semantics.
5. Confirm C / C_eff still consumes the curve-gap context this function builds (after
   Prompt 1 if already merged).

## Change
1. In Adds’ curve-bucket construction, **count the commander’s CMC** into the appropriate
   bucket, using the **same inclusion rules Cuts already uses** (same CMC value source,
   same X handling if Cuts has one, same commander object lookup).
2. Prefer a shared helper if Cuts/Adds already share one and Adds simply skips the
   commander argument — fix by passing/including commander consistently.
3. If Adds duplicates Cuts’ loop and omits commander: add the commander CMC bucket step
   to match Cuts line-for-line in behavior (not necessarily copy-paste structure).
4. Leave scoring-term formulas untouched; only the curve context feeding C changes.

## Do NOT touch
- Cuts curve logic (beyond reading it as the behavioral reference)
- Other Adds scoring terms / constants from Prompt 1 (7 / 9 / 10 / 11 / 12)
- Entry 13 plan wizard / plan schema / Entry 5 backfill gate
- Entry 6 owned/all-cards candidate pool
- `tribes: []`, `CK_REQUIRED_ENABLERS`
- Candidate pool filters, owned-first sort, top-8 count
- Live Scryfall / EDHREC scrape
- Runtime AI/LLM

## Verification

### Hard (must pass)
| # | Case | Expected |
|---|------|----------|
| 1 | Commander CMC = 4, non-land list otherwise identical | Adds curve bucket for CMC 4 is **+1** vs pre-fix (commander counted once) |
| 2 | Same deck, Compare Adds vs Cuts curve buckets for non-land CMCs | **Commander CMC bucket matches** Cuts’ inclusion of commander (same count contribution) |
| 3 | High-CMC commander (e.g. 6+) on a curve short at that slot | C / C_eff for a candidate filling that slot **moves** vs pre-fix in the direction implied by the corrected gap (document before/after term log) |

### Soft (PR write-up)
| # | Case | Expectation |
|---|------|-------------|
| 4 | Partner already merged Prompt 1 (C_eff / L) | C_eff still uses corrected buckets; L / efficiency-mode tagging unchanged |
| 5 | Token / land / multi-faced edge cases | Follow Cuts’ existing treatment; document any remaining intentional asymmetry |

**Verification delivery:** before/after curve-bucket dump (debug or test) for at least one
fixed test deck; assert commander CMC counted once on Adds; note line anchors found in
Step 0.

## Deliverables
- Adds curve-bucket construction includes commander CMC, matching Cuts behavior
- No Cuts / other scoring-term edits
- Step 0 findings (anchors + whether shared helper existed) in PR notes
- Hard cases 1–3 evidenced (test or logged before/after)
```

---

# Prompt 4 of 4 — Collection / All Cards pool toggle (entry 6)

```
# Adds pool toggle — Collection / All Cards (entry 6)

**Prereq:** Prefer Prompt 1 merged (better All Cards rankings). Prefer Prompt 2 merged
so Entry 5's plan-aware hybrid backfill exists before this prompt replaces the hybrid
owned→backfill gathering UX with an explicit toggle. **Do not start this before Prompt 2
if Prompt 2 is still in flight** — Entry 6 must not rewrite Prompt 2's wizard/schema/Entry 5
gate text; it layers pool modes afterward. Safe to parallel Prompt 3 (commander CMC curve).

## Hard constraint
Deterministic algorithm only — no runtime LLM, embeddings, or other AI/ML inference.

## Context
Today Adds is owned-focused: owned collection first, then deficit-gated unowned backfill
from `/api/cards/by-roles` (local DB only; never live Scryfall; `tribes: []`).

Entry 6 replaces that hybrid gathering UX with an explicit Adds-panel toggle:

| UI label | Pool behavior |
|----------|----------------|
| **Collection** | Owned collection only. **Never** call server backfill, regardless of deficits. |
| **All Cards** | Always score **full local DB ∩ format + commander color identity** (plus existing "not already in deck" / free-copy style filters). **No** live Scryfall. **No** deficit-gated server backfill in this mode. Rank by **score only** (no owned-first). |

Verify line anchors before editing (may have drifted):
- `_renderAddSuggestions` (~decks.js:6623) — pool assembly, owned-first, backfill gate
- Unowned / Plan-only deficit gate (~decks.js:6679) — Prompt 2 may have changed this
- `/api/cards/by-roles` (or successor) — backfill endpoint (local DB)
- Existing user preference / settings persistence patterns (for toggle memory)

## Locked design decisions (Entry 6 interview — do not re-open)
1. All Cards = local DB catalog only (not live Scryfall; not "every printed card offline").
2. All Cards ranks by score only — disable owned-first / owned boost.
3. Remember last choice; first-ever visit defaults to **Collection**.
4. Persist preference: **server-synced per-user** if the app already has that pattern;
   else **per-user global** client persistence. Not per-deck. Not session-only.
5. Do **not** change Prompt 1 scoring terms or Prompt 2 wizard/Entry 5 hybrid gate
   as a dependency of this work. This prompt implements the toggle layer; if Prompt 2
   already shipped, Collection simply never backfills and All Cards uses the full local
   pool (Entry 5's Plan-only fetch gate may become unused while All Cards is active —
   document that; do not rip out plan schema / wizard / planMatchScore).
6. UI labels: **Collection** / **All Cards**.

## Goal
Ship the Collection / All Cards toggle on the Adds panel with the pool + persistence
behaviors above. Candidate-pool change only — do not retune scoring formulas.

## Step 0 — Repo discovery (do first; document in PR)
1. Trace how Adds currently builds the candidate list (owned filter, owned-first sort,
   deficit gate, `/api/cards/by-roles` backfill).
2. Confirm local DB access path for "all format + CI legal cards" without live Scryfall.
3. Find existing **user preference** persistence (server-synced settings vs localStorage).
   Choose D-then-A per interview #4; document which path you used and why.
4. Confirm Prompt 2 status in this repo: if Entry 5 gate exists, note how Collection /
   All Cards should interact without deleting wizard/plan fields.
5. Confirm color-identity + "not in deck" filters still apply in both modes.

## Change

### UI
- Adds panel control with two modes labeled **Collection** and **All Cards**.
- Switching modes recomputes suggestions with the matching pool rules.
- Persist selection per interview #3–4; first paint = Collection until a stored choice
  exists.

### Collection mode
- Candidate pool = owned collection only (existing ownership / free-copy rules).
- **Do not** call unowned / server backfill for any deficit state (including Plan-only).
- Sorting within the owned pool: existing score order is fine (owned-first is redundant
  when every candidate is owned).

### All Cards mode
- Candidate pool = local DB ∩ format legality ∩ commander color identity ∩ not already
  in deck (and any other existing non-ownership legality filters).
- **Do not** use deficit-gated backfill to grow the pool — the local DB **is** the pool.
- **Do not** apply owned-first sort or owned boost; sort by Adds score only (top 8).
- Still never live Scryfall. Keep `tribes: []` if any residual by-roles call remains;
  prefer not calling backfill at all in this mode.

### Scoring / gates to preserve
- Reuse `_scoreAddCandidate` (and Prompt 1 terms if present) unchanged.
- Keep `CK_REQUIRED_ENABLERS` hard gate.
- Keep top 8 (`_ADD_SUGGESTION_COUNT`).
- Do not touch Cuts.

## Do NOT touch
- Prompt 1 scoring formulas/constants (except consuming them as-is)
- Prompt 2 deck-plan wizard, plan schema, or Entry 5 gate implementation details —
  do not reopen or rewrite that prompt; only adapt pool gathering for the new modes
- Cuts scoring / Cuts UI
- Entry 1 curve-bucket logic (Prompt 3) unless an unavoidable shared helper conflict
  appears — prefer not
- Live Scryfall / EDHREC scrape
- `tribes: []` intentional empty send
- Runtime AI/LLM

## Verification

### Hard (must pass)
| # | Case | Expected |
|---|------|----------|
| 1 | Mode = Collection, Plan-only or role deficit | Suggestions ⊆ owned; **no** `/api/cards/by-roles` (or successor) backfill call |
| 2 | Mode = All Cards | Top 8 may include unowned local-DB cards; order is score-only (owned card must not outrank higher-scoring unowned solely due to ownership) |
| 3 | First-ever user (no saved pref) | Control initializes to **Collection** |
| 4 | User selects All Cards, reloads Adds | Mode restores to **All Cards** (synced pref if available, else per-user global) |
| 5 | All Cards pool | Every suggestion is format + color-identity legal and from local DB (no live Scryfall) |

### Soft (PR write-up)
| # | Case | Expectation |
|---|------|-------------|
| 6 | Prompt 2 Entry 5 already merged | Wizard/plan fields still work; document how Collection/All Cards relate to the old hybrid backfill gate |
| 7 | Large local DB | All Cards remains usable (note any pagination/caps you needed; do not silently fall back to owned-only) |

## Deliverables
- Adds UI toggle: **Collection** / **All Cards**
- Pool behavior per mode (including no backfill in Collection; full local DB in All Cards)
- Preference persistence (server-synced per-user if possible, else per-user global)
- Step 0 findings (anchors, prefs path chosen, Prompt 2 interaction notes)
- Hard cases 1–5 evidenced in PR
```

---

# Prompt 5 of 5 — Adds token exclusion (entry 2)

```
# Adds — exclude tokens from Plan-count; never recommend tokens (entry 2)

## Context
**Confirmed bug** (project quirk #1): Cuts excludes token cards from its candidate pool, so
tokens never enter Plan-count math on the Cuts side. Adds' Plan-count / deficit logic does not
mirror that exclusion, and Adds may surface token cards as suggestions.

**User decision (2026-07-14):** Token cards should **never** count toward Plan and should
**never** be recommended — tokens are byproducts of other cards' abilities, not real 99
slots. **Token generators** (regular non-token cards that create tokens) are **different**
and must not be conflated.

Verify line anchors before editing (may have drifted):
- Adds Plan-count / deficit logic (~decks.js:6294) in `_computeAddContext` (~6274)
- Adds candidate pool assembly (~decks.js:6623 `_renderAddSuggestions`)
- Cuts token exclusion (~decks.js:6254 `_suggestCardsToCut`) — **read-only reference**
- Any shared `isToken` / card-type helper Cuts already uses

## Goal
1. **Plan-count:** Adds excludes **token cards** from Plan tally — same predicate Cuts uses.
2. **Recommendations:** Adds must **never** recommend token cards (owned, catalog, backfill).
3. **Token generators stay:** non-token cards that create tokens remain valid candidates and
   normal role/Plan math.

**Hard constraint:** Deterministic algorithm only — no runtime AI/LLM/ML inference.

## Locked design decisions (do not re-open)
- Match Cuts' token detection (`isToken` / type line) — **not** oracle "creates tokens" text.
- **Adds-only** — do not change Cuts.
- Token generators (e.g. Parallel Lives, Young Pyromancer) are **in scope as normal cards**.
- Only **token-type cards** are excluded from Plan-count and from the Adds candidate pool.

## Step 0 — Repo discovery (do first; document in PR)
1. Locate Cuts' token exclusion predicate — reuse or share it; do not invent a second rule.
2. Read Adds Plan-count path in `_computeAddContext` — confirm tokens are currently counted.
3. Audit Adds candidate gathering (owned, local DB / catalog, `/api/cards/by-roles` backfill):
   confirm whether token cards can appear today; note every path that must filter them out.
4. Audit other Adds "deck cards for recipe" tallies — apply consistent token exclusion if any
   other count mirrors Plan logic.
5. Document one token card and one token-generator card from the local DB for verification.

## Change

### Plan-count
1. In Adds' Plan-count / roleless-card tally, **skip token cards** using the same helper/rule
   Cuts uses for `commander/tokens/lands` exclusion.
2. Prefer extracting a shared `isTokenCard(card)` (or reusing existing) if duplicated.

### Candidate pool
3. Ensure **every** Adds suggestion path filters out token cards before scoring/ranking:
   owned collection, full local DB (Entry 6 All Cards if present), and server backfill results.
4. Filter at pool-build time (drop tokens), not only at display time.

## Do NOT touch
- Cuts scoring / Cuts candidate pool (read as reference only)
- Token **generator** cards — must remain suggestable
- Scoring formulas (D, M, C, L, E, B, P, V, T, K) and weights
- Entry 13 plan wizard / plan schema (except Plan deficit now excludes tokens correctly)
- `tribes: []`, `CK_REQUIRED_ENABLERS`
- Live Scryfall / EDHREC scrape
- Runtime AI/LLM

## Verification

### Hard (must pass)
| # | Case | Expected |
|---|------|----------|
| 1 | Deck with 5 untagged **token** cards + 35 other untagged non-tokens | Adds Plan count = **35** (tokens excluded), matching Cuts-side semantics |
| 2 | Same deck, Plan deficit active | Adds Plan deficit **≥** pre-fix (tokens no longer inflate Plan count) |
| 3 | Adds suggestion run (any pool mode) | **No** suggested card is a token (type-line token / `isToken`) |
| 4 | Deck with token **generator** (e.g. Parallel Lives) not in deck | Generator **can** appear in Adds suggestions when it scores highly |
| 5 | Token generator in deck, untagged | Counts toward Plan (or role tags) like any other non-token card |

### Soft (PR write-up)
| # | Case | Expectation |
|---|------|-------------|
| 6 | Prompt 2 Entry 13 merged | Plan-only backfill uses corrected Plan deficit (tokens not suppressing fetch) |
| 7 | Zero token cards in 99 | Behavior unchanged vs today |

## Deliverables
- Shared or mirrored token predicate (document anchor)
- Plan-count fix in `_computeAddContext` (or equivalent)
- Candidate-pool token filter on all Adds gather paths
- Step 0 findings + hard cases 1–5 evidenced in PR
```

---

*End of ready-prompts catalog. Add new Prompt drafted items here in queue order; remove when Shipped.*
