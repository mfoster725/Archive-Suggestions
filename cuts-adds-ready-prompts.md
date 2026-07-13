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

### Not in this doc yet (not Prompt drafted)

| Entry | Status | Note |
|-------|--------|------|
| 1 — Commander CMC in Adds curve | Fix scoped | Ready for a prompt when you ask; intentionally separate from #1 above unless you bundle. |
| 6 — Owned/All Cards toggle | Fix scoped (partial) | Owned mode only; All Cards data source unresolved. |
| 13 v2 / Cuts plan / hybrid modifiers | Design only | After 13 v1 ships. |

---

## How to use

1. Open the **main app repo** (partner) in Cursor / cloud agent.
2. Copy **one** fenced prompt block below (start at `# …` inside the fence).
3. Paste into the agent. Say start / implement.
4. After merge, mark that backlog entry **Shipped** and move full write-up to
   `cuts-adds-archive.md`; remove or strike that prompt from this file.

---

# Prompt 1 of 2 — Coordinated Adds scoring rebalance (entries 7 / 9 / 10 / 11 / 12)

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
Implement coordinated scoring changes so ALL verification cases pass (see bottom).

**Hard constraint:** Deterministic algorithm only — no runtime AI/LLM.

## Step 0 — Repo discovery (do this first, document in PR)
1. Read `_scoreAddCandidate` — confirm current D, M, C, V, T, K math and constants.
2. Locate project **role-tag IDs/names** (~36 utility tags). Build constants from semantic
   lists in backlog entry 11 — do NOT assume Scryfall `otag:` slugs match project IDs.
3. Locate **archetype detection** for spellslinger (or equivalent). Document function used
   for entry 12 B-term gating.
4. Confirm `edhrec_rank` and USD price available on card objects in local DB.
5. Log term breakdown helper for verification (debug flag or unit test).

## Term changes

### D — sublinear multi-deficit scaling (entry 10, PRIMARY)
When candidate matches multiple active deficits:
- Collect matched deficit magnitudes; sort descending.
- `D = Σ deficit_i × weight_i` where weights = `[1.0, 0.40, 0.20]` for 1st/2nd/3rd+.
- Single-deficit candidates: unchanged (weight 1.0 only).

### L + C_eff — CMC efficiency for interaction roles (entry 11)
Build `EFFICIENCY_MODE_PROJECT_TAGS` from backlog entry 11 Tier 1 + Tier 2 semantic
categories mapped to project tag IDs. Exclude lands from L.

If candidate has ≥1 efficiency-mode tag AND is not a land:
- `C_eff = 0`
- `L = K_L × max(0, CMC_REF − CMC)` with `CMC_REF = 4`, `K_L` tuned
Else:
- `C_eff = C` (existing curve-gap bonus)
- `L = 0`

Do NOT apply L / do not zero C for: Board Wipe, Card Draw (general), draw engines,
Plan/untagged, land-ramp categories — see backlog entry 11 exclusion table.

### E — price-aware EDHREC percentile (entry 7)
Precompute server-side (never live per suggestion):
- Per role tag, store percentile from `edhrec_rank` within that tag's ranked population.
- Min population 8; below → E = 0 (neutral).
- **Price-aware:** adjust rank/percentile so expensive staples aren't systematically
  underrated (document formula; Three Visits rank ~42 must stay elite).

At scoring: **one E per candidate** using percentile for **largest active deficit's role**.
Multi-tag dampening inside E only (optional). Do NOT sum E per tag.
Do NOT use EDHREC category APIs or scrape edhrec.com.

### B — creature body bonus (entry 12)
If `!deckIsSpellslinger(deck)` AND candidate is Creature AND fills active deficit:
- `B = K_B_RAMP` when ramp deficit active and candidate has Ramp tag
- else `B = K_B` for other qualifying creatures (v1: ramp-focused; extend later)
Else `B = 0`.

Must flip: Sakura-Tribe Elder > Rampant Growth; Wood Elves > Rampant Growth even though
L favors RG by 1 point (CMC 2 vs 3). Calibrate K_B_RAMP accordingly.

### P — colored pip restrictiveness (entry 9)
`P = K_P × pip_restrictiveness_score`, computed from parsed mana cost:
- W/U/B/R/G pips: full weight (1.0 each)
- Hybrid pips (e.g. `{G/U}`): dampened, ~0.5 weight each
- Phyrexian pips (e.g. `{G/P}`): **negative weight** (small bonus, not neutral) — more
  flexible than colorless since payable with life; exact magnitude TBD, calibrate with
  other P constants
- Colorless `{C}`: 0 weight
- Generic `{1}`/`{2}`/etc.: 0 weight
- `{X}`: 0 weight for P (no color symbol)

Subtract `P` from total. Penalize regardless of on-color status.

**Effective CMC convention (applies to C_eff/L, not P):** treat `{X}` as **X = 3** for
any CMC-based term. Apply this consistently wherever CMC-based scoring reads a card's CMC.

### V — dampen multi-tag versatility (entry 10, tertiary)
Keep V positive. Dampen for 2+ utility tags (~50% on 2nd+ tag contribution).
Do NOT add Cuts-style subtractive multi-role discount on total score.

## Final formula
`Score = (D × M) + C_eff + L + E + B − P + V + T + K`

## Weight order (calibration guide)
`D, M` > `C or L` > `E` > `B` > `P` > `V` > `T, K`

## Do NOT touch
- Cuts / `_suggestCardsToCut`
- Adds candidate pool / owned vs backfill (entry 6)
- `tribes: []` on backfill (intentional)
- `CK_REQUIRED_ENABLERS` (15)
- Entry 1 commander CMC curve fix (unless user says bundle)
- Entry 13 plan wizard (prompt 2 — run after this ships)

## Verification (all required)
| # | Case | Expected winner |
|---|------|-----------------|
| 1 | Simic, ramp deficit > draw deficit | Three Visits > Growth Spiral |
| 2 | Ramp deficit active | Three Visits > Cultivate |
| 3 | Non-spellslinger green ramp deck | Sakura-Tribe Elder > Rampant Growth |
| 4 | Same deck | Wood Elves > Rampant Growth |
| 5 | Board-wipe deficit only | Sweepers still get C (L not applied) |
| 6 | Spellslinger deck (if detectable) | B = 0; RG may beat STE |
| 7 | Term isolation | E favors TV over GS in ramp context but cannot alone flip #1 |

Log D, M, C_eff, L, E, B, P, V, T, K for each verification pair.

## Deliverables
- Code + named constants (`D_SUBLINEAR_WEIGHTS`, `K_L`, `CMC_REF`, `K_P`, `K_B`, `K_B_RAMP`, E params)
- `EFFICIENCY_MODE_PROJECT_TAGS` with mapping comment
- Formula comment block in `_scoreAddCandidate`
- Precompute job/migration for E percentiles if not present
- Test or debug output for all 7 verification cases
```

---

# Prompt 2 of 2 — Entry 13 v1 + Entry 5 (plan wizard + plan-aware backfill)

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
  "fieldSources": {
    "winConditionId": "chip-confirmed",
    "primaryStrategyId": "formal",
    "secondaryStrategyId": null
  },
  "tertiaryStrategyId": null,
  "hybridRoleModifiers": null,
  "cutsShielding": null
}
- Required for "plan declared": winConditionId + primaryStrategyId
- secondaryStrategyId optional / skippable
- fieldSources: chip-confirmed | chip-corrected | formal | skipped→formal
- Last three fields: v2 hooks — nullable, unused in v1

## Wizard questions

All multiple choice. Stable IDs from catalogs. Show More Options. All skippable.
User can navigate back and edit any prior answer anytime.

### Path A — deckCardCount < 80
1. Commander — confirm/set if missing
2. Win condition — "How does this deck usually win?"
   Top 6 from rankWinConditionsForCommander, else static fallback; full catalog in Show More
3. Primary strategy — "What is the main strategy or theme?"
   Top 6 from rankStrategiesForCommander, else static fallback; full catalog in Show More
4. Secondary strategy (optional) — skippable

### Path B — deckCardCount >= 80
1. Run rankStrategiesForDeck + rankWinConditionsForDeck (+ archetype hint)
2. At most 3 chips: suggested wincon, primary strategy, optional archetype
   (chip only if score >= PLAN_INFERENCE_CONFIDENCE_MIN)
3. Per chip Confirm / Correct / Skip
   - Confirm or Correct → skip corresponding formal Q
   - Skip or missing → formal Q; pre-fill if score >= min
4. Correct opens SAME shared picker as formal Q (including Show More)
5. Optional secondary strategy at end

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

## Named constants
PLAN_WIZARD_ANALYZE_THRESHOLD = 80
PLAN_PRIMARY_OPTIONS_COUNT = 6
PLAN_INFERENCE_CONFIDENCE_MIN = 0.35
PLAN_CHIP_MAX = 3
PLAN_TAG_SIGNAL_WEIGHT = 1.0
PLAN_ORACLE_SIGNAL_WEIGHT = 0.5

PLAN_INFERENCE_CONFIDENCE_MIN is a normalized 0–1 match-score cutoff (not "35% feature
confidence"). Below 0.35 → static fallback; do not trust chips/pre-fill.

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

## Answers → data
MC pick → store ID on deck (winConditionId / primaryStrategyId / secondaryStrategyId).
No free-text parse. Labels are UI; IDs are scoreable.

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

Log inference scores, chip actions, fieldSources, planMatchScore.

## Deliverables
- Schema + persistence (+ v2 nullable hooks)
- Wizard Path A and Path B
- Shared picker + Show More + back nav
- rankForCommander / rankForDeck + named constants
- Semantic→project tag ID map in code
- Entry 5 gate + planMatchScore
- Archetype ignored on plan-backfill when plan declared
- Cases 1–7 evidenced in PR
- Step 0 findings in PR notes

## Build order inside this prompt
1. Schema
2. Path A wizard
3. Plan backfill (prove Entry 5 early)
4. Path B chips + shared picker
5. Optional secondary + polish
```

---

*End of ready-prompts catalog. Add new Prompt drafted items here in queue order; remove when Shipped.*
