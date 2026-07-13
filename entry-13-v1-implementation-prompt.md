# Entry 13 v1 — Implementation prompt (copy this entire file to an agent)

**Status:** Prompt drafted (2026-07-13). Ready to run in the **main deck-builder app repo**
(the one that contains `decks.js`). Do **not** run this in Archive-Suggestions (docs only).

**Hard constraint:** Deliver **deterministic algorithm code only**. No runtime LLM, embeddings,
or other AI/ML inference. Multiple-choice answers, lookup tables, keyword rules, and formulas
only. Coding agents write the code; the shipped product does not call AI.

**Source of truth for design decisions:** `cuts-adds-backlog.md` Entry 13 (interview #20–31,
v1 MVP, v1 algorithm spec). This prompt is self-contained for implementation.

---

## Goal

Ship **Entry 13 v1**: a guided **deck-plan wizard** that stores structured plan data, plus
**plan-aware Adds backfill** (Entry 5) so Plan-only deficits fetch on-theme cards instead of
stalling or returning generic untagged cards.

**Out of scope (v2 — do not implement):**
- Hybrid functional-role weight modifiers
- Cuts plan-awareness / shielding
- Tertiary strategy slot
- Beginner/Intermediate/Advanced wording variants
- Free-text plan notes
- Multi-role recommendation explanation UI
- Large commander affinity DB / catalog expansion beyond v1 tables below

---

## Step 0 — Repo discovery (do first; document in PR)

Archive-Suggestions has **no** `decks.js`. Find the real app files before editing.

1. Locate `decks.js` (or equivalent). Verify / update these anchors (may have drifted):
   - Adds unowned / Plan-only deficit gate (~6679)
   - `_renderAddSuggestions` (~6623)
   - `_scoreAddCandidate` (~6489)
   - Plan-count / Plan deficit logic (~6294)
   - `_computeAddContext` (~6274)
   - `_suggestCardsToCut` (~6254) — **read only; do not change Cuts**
2. Find where deck JSON / metadata is persisted. Add plan fields there.
3. Find existing **archetype** detection / override. Document how plan will override it for
   the plan-backfill path only.
4. Enumerate project **role-tag IDs/names** (~36 utility tags). Map every semantic signal
   in this prompt to **real project tag IDs** — do not assume Scryfall `otag:` slugs match.
5. Find `/api/cards/by-roles` (or successor). Note how to extend filters for plan-aware
   Plan backfill without live Scryfall.
6. Find existing modal / wizard / multi-step UI patterns; match them.
7. Confirm deck card count includes non-lands as used today for “deck size” thresholds;
   document the exact count used for `PLAN_WIZARD_ANALYZE_THRESHOLD`.

---

## Current behavior (verify, then change)

- Recipe includes **Plan 30** as “cards with no utility role tag.”
- Unowned Adds backfill requires a **non-Plan** deficit. Plan-only deficit → no fetch →
  likely “no suggestions” on role-healthy theme-short decks (Entry 5).
- “Plan” has **no positive definition** — no wizard, no declared strategy/win condition.
- Archetype + Aggro↔Control slider adjust recipe today; Entry 13 must not rewrite Cuts.

---

## Desired v1 behavior

### Plan schema (persist on deck)

```json
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
```

- **Required for “plan declared”:** `winConditionId` + `primaryStrategyId`
- `secondaryStrategyId` optional / skippable
- `fieldSources` values: `chip-confirmed` | `chip-corrected` | `formal` | `skipped→formal`
- Last three fields: **v2 hooks** — nullable, unused in v1

### Wizard questions (exact flow)

All questions are **multiple choice**. Every option has a stable ID from the catalogs below.
Each question has **Show More Options**. All questions are **skippable**. User can
**navigate back** and edit any prior answer at any time.

#### Path A — `deckCardCount < 80`

1. **Commander** — Confirm or set commander if missing. (Skip if already set.)
2. **Win condition** — “How does this deck usually win?”  
   Options: top 6 from `rankWinConditionsForCommander(commander)`, else static fallback;
   full catalog under Show More.
3. **Primary strategy** — “What is the main strategy or theme?”  
   Options: top 6 from `rankStrategiesForCommander(commander)`, else static fallback;
   full catalog under Show More.
4. **Secondary strategy (optional)** — “Any secondary theme?” Skip allowed.

Completion for scoring/backfill: win condition + primary strategy set (or user exits early
without a complete plan — backfill gate stays closed).

#### Path B — `deckCardCount >= 80`

1. Run `rankStrategiesForDeck` + `rankWinConditionsForDeck` (+ existing archetype hint).
2. Show **at most 3 observation chips**:
   - Suggested win condition (rank #1 if score ≥ `PLAN_INFERENCE_CONFIDENCE_MIN`)
   - Suggested primary strategy (same rule)
   - Optional archetype hint (existing system)
3. Per chip: **Confirm** / **Correct** / **Skip**
   - Confirm or Correct → field locked; **skip corresponding formal question**
   - Skip or missing chip → run formal question; **pre-fill** from inference if score ≥ min
4. **Correct** opens the **same** shared picker as the formal question (including Show More).
5. Optional secondary strategy question once at the end.

### Catalogs (use these exact IDs)

#### Strategies (15)

| ID | Label |
|----|-------|
| `strategy.tokens` | Tokens / Go-wide |
| `strategy.sacrifice` | Sacrifice / Aristocrats |
| `strategy.spellslinger` | Spellslinger |
| `strategy.reanimator` | Reanimator / Graveyard |
| `strategy.voltron` | Voltron / Commander damage |
| `strategy.counters` | +1/+1 Counters |
| `strategy.landfall` | Landfall |
| `strategy.tribal` | Tribal |
| `strategy.artifacts` | Artifacts |
| `strategy.enchantress` | Enchantress |
| `strategy.control` | Control / Value grind |
| `strategy.blink` | Blink / ETB value |
| `strategy.superfriends` | Superfriends |
| `strategy.theft` | Theft / Steal |
| `strategy.other` | Other / Hybrid |

**Static fallback top 6:** `strategy.tokens`, `strategy.sacrifice`, `strategy.spellslinger`,
`strategy.tribal`, `strategy.control`, `strategy.other`

#### Win conditions (8)

| ID | Label |
|----|-------|
| `wincon.combat` | Combat damage (combat wins) |
| `wincon.commander_damage` | Commander damage |
| `wincon.combo` | Infinite / instant-win combo |
| `wincon.mill` | Mill |
| `wincon.life_drain` | Life drain / life loss |
| `wincon.lock` | Lock / Stax / hard lock |
| `wincon.value` | Overwhelming value / grind |
| `wincon.other` | Other |

**Static fallback top 5:** `wincon.combat`, `wincon.commander_damage`, `wincon.combo`,
`wincon.life_drain`, `wincon.value`

### Named constants

| Constant | Value |
|----------|-------|
| `PLAN_WIZARD_ANALYZE_THRESHOLD` | `80` |
| `PLAN_PRIMARY_OPTIONS_COUNT` | `6` |
| `PLAN_INFERENCE_CONFIDENCE_MIN` | `0.35` |
| `PLAN_CHIP_MAX` | `3` |
| `PLAN_TAG_SIGNAL_WEIGHT` | `1.0` |
| `PLAN_ORACLE_SIGNAL_WEIGHT` | `0.5` |

**About `PLAN_INFERENCE_CONFIDENCE_MIN`:** This is a **normalized match score cutoff**
(0–1), **not** “35% confident the feature works.” If the top-ranked option’s score is
**below 0.35**, use the static fallback list and do not treat inference as trustworthy
for chips / pre-fill.

### Ranking algorithms (deterministic)

#### `rankForCommander(commander)` — Path A

Case-insensitive substring match on commander oracle text. Each hit adds
`PLAN_ORACLE_SIGNAL_WEIGHT` (cap 3 hits per catalog ID). Return top
`PLAN_PRIMARY_OPTIONS_COUNT`. If top score &lt; `PLAN_INFERENCE_CONFIDENCE_MIN`, return
static fallback.

**Strategy keyword → ID boosts:**

| Keywords | Strategy ID |
|----------|-------------|
| sacrifice, sacrifices, dies | `strategy.sacrifice` |
| token, tokens | `strategy.tokens` |
| cast, instant, sorcery, magecraft, storm | `strategy.spellslinger` |
| graveyard, reanimate (and “return” near “graveyard”) | `strategy.reanimator` |
| commander damage, equipped, aura | `strategy.voltron` |
| +1/+1 counter, proliferate | `strategy.counters` |
| landfall, land enters | `strategy.landfall` |
| tribal / clear single creature-type identity | `strategy.tribal` |
| artifact | `strategy.artifacts` |
| enchantment | `strategy.enchantress` |
| counter target, draw (control-leaning) | `strategy.control` |
| flicker, exile+return, enters the battlefield | `strategy.blink` |
| planeswalker, loyalty | `strategy.superfriends` |
| gain control, steal | `strategy.theft` |

**Win-condition keyword → ID boosts (weaker; expect more fallbacks):**

| Keywords | Wincon ID |
|----------|-----------|
| mill, library→opponent graveyard cues | `wincon.mill` |
| lose life, drain, lifelink | `wincon.life_drain` |
| infinite, win the game, you win | `wincon.combo` |
| can't / prevent / skip phase (staxy) | `wincon.lock` |
| commander damage | `wincon.commander_damage` |
| combat damage | `wincon.combat` |

#### `rankForDeck(deck)` — Path B

Only when card count ≥ 80.

1. Build deck signal vector from **role-tag counts** (project IDs) and card-type ratios
   (creatures, instants/sorceries, artifacts, enchantments, lands).
2. Score each strategy/wincon from signal tables below × `PLAN_TAG_SIGNAL_WEIGHT`.
3. Normalize scores to 0–1 by dividing by that catalog’s max attainable score.
4. Return top 6; if top &lt; `PLAN_INFERENCE_CONFIDENCE_MIN` → static fallback.
5. Chip value = rank #1 only when score ≥ min; else omit chip (user must pick).

**Strategy deck signals (semantic — map to project tag IDs in Step 0):**

| Strategy ID | Signals |
|-------------|---------|
| `strategy.tokens` | token-producing, go-wide |
| `strategy.sacrifice` | sacrifice outlets, dies triggers, aristocrats |
| `strategy.spellslinger` | instant/sorcery density, cast triggers, prowess |
| `strategy.reanimator` | recursion, reanimate |
| `strategy.voltron` | equipment, auras, commander-centric |
| `strategy.counters` | +1/+1 counters, proliferate |
| `strategy.landfall` | landfall, land recursion |
| `strategy.tribal` | dominant creature type share &gt; ~40% |
| `strategy.artifacts` | artifact density / synergy |
| `strategy.enchantress` | enchantment density / enchantress |
| `strategy.control` | counterspell, removal, draw density |
| `strategy.blink` | ETB, flicker |
| `strategy.superfriends` | planeswalker count |
| `strategy.theft` | steal / gain control |

**Wincon deck signals (sparse in v1):**

| Wincon ID | Signals |
|-----------|---------|
| `wincon.combat` | high creature count, combat keywords |
| `wincon.commander_damage` | voltron signals |
| `wincon.combo` | tutors + combo enablers |
| `wincon.mill` | mill |
| `wincon.life_drain` | drain, lifelink, ping |
| `wincon.lock` | stax / prison / resource denial |
| `wincon.value` | high draw + removal grind |

### Plan-aware backfill (Entry 5)

**Gate:** Allow unowned fetch when the largest active deficit is **Plan** AND the deck has
both `winConditionId` and `primaryStrategyId` set. If plan not declared, keep current
behavior (no Plan-only unowned fetch).

**Match score:**

```
planMatchScore(card) =
  2 * strategyMatch(card, primaryStrategyId)
  + 1 * strategyMatch(card, secondaryStrategyId)   // if set, else 0
  + 1 * winconMatch(card, winConditionId)
```

`strategyMatch` / `winconMatch` ∈ {0, 1+} using the same keyword/tag vocabulary as ranking
tables, mapped to project tags / oracle / types in Step 0. Document the lookup tables in
code as named constants.

**Rank:** Within the Plan backfill pool, sort by `planMatchScore` desc, then existing Adds
score. **Equal-weight Plan role** — do not add hybrid role-weight modifiers.

**Archetype:** When plan fields are set, do **not** use archetype for this plan-backfill
filter/rank path (declared plan overrides). Do not rewrite the entire recipe engine in v1.

### How wizard answers become usable data

| User action | Stored field | Consumed by |
|-------------|--------------|-------------|
| Pick win condition | `winConditionId` | ranking memory, backfill |
| Pick primary strategy | `primaryStrategyId` | ranking memory, backfill (×2) |
| Pick secondary strategy | `secondaryStrategyId` | backfill (×1) |
| Chip confirm/correct | same fields + `fieldSources` | skips re-ask; authoritative |

No free-text interpretation. Option label is UI; **ID** is the scoreable datum.

---

## Do not touch

- **Cuts** scoring / shielding (v2)
- Hybrid role-weight modifiers (v2)
- Intentional quirks: `tribes: []` on Adds backfill; CK enabler hard gate
- Entries 7–12 scoring terms (unless already shipped and only consumed as-is)
- Live Scryfall or EDHREC scraping
- Any runtime AI/LLM

---

## Verification (must pass / demonstrate)

| # | Scenario | Expected |
|---|----------|----------|
| 1 | ≥80-card sacrifice/aristocrats deck | Chips suggest `strategy.sacrifice` and a sensible wincon when score ≥ 0.35; confirm skips formal Q |
| 2 | &lt;80 cards, Korvold (or sacrifice-heavy commander) | `strategy.sacrifice` in top 6 from oracle rules |
| 3 | Plan-only deficit + plan declared | Unowned fetch runs; results show elevated `planMatchScore` for on-theme plan cards |
| 4 | Plan-only deficit + plan **not** declared | No unowned fetch (unchanged) |
| 5 | All inference scores &lt; 0.35 | Static fallback lists; no overconfident chip |
| 6 | User skips chip | Formal Q runs, pre-filled if score ≥ 0.35 |
| 7 | User goes back and edits | Prior answers change; schema updates; later answers preserved |

Add debug logging for: inference scores, chip actions, `fieldSources`, `planMatchScore`.

---

## Deliverables checklist

- [ ] Plan schema persisted on deck (+ v2 nullable hooks)
- [ ] Wizard UI Path A and Path B
- [ ] Shared picker + Show More + back navigation
- [ ] `rankForCommander` / `rankForDeck` with named constants
- [ ] Semantic→project tag ID mapping table documented in code
- [ ] Entry 5 Plan-only backfill gate + `planMatchScore` filter/rank
- [ ] Archetype ignored on plan-backfill path when plan declared
- [ ] Verification cases 1–7 evidenced in PR (screenshots, logs, or tests)
- [ ] PR notes: Step 0 findings (real line numbers, tag ID map)

---

## Suggested implementation order

1. Schema + persistence  
2. Path A wizard (commander → wincon → primary)  
3. Plan backfill gate + `planMatchScore` (**prove Entry 5 payoff early**)  
4. Path B analysis + chips + shared picker  
5. Optional secondary + polish + verification log  

---

*End of prompt — copy everything above this line to the implementing agent.*
