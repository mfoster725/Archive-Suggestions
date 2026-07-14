# Cuts/Adds Improvement Backlog (working doc)

This is a living document, separate from the fixed project instructions. Its job is to
track scoped, individually actionable items for the Cuts/Adds feature — both fixes for
"the suggestions are bad" and enhancements/new capabilities the feature doesn't have yet.
Each item should be small enough to become a single agent prompt later. An entry doesn't
need a bad-suggestion symptom to be valid — a feature request with a clear proposed
behavior is just as legitimate an entry as a bug.

**Archive:** Closed/ruled-out/shipped entries with full reasoning live in
`cuts-adds-archive.md`, referenced here by ID with a one-line summary. Pull the archive
into context only if an entry needs to be re-litigated.

**Partner knowledge layer (WIP, not done):** See
[`partner-knowledge-layer-phases.md`](./partner-knowledge-layer-phases.md). Partner is
building card-axis extraction + Phases 3–5 (interactions → goals → recommender). That work
**complements** this backlog's Suggested Adds/Cuts improvements; it is not shipped and is
not a ready-prompt queue item here.

## Note for coding agents (read this first)

**This project builds a deterministic algorithm, not an AI product.**

- The Cuts/Adds recommendation engine — including Entry 13's deck-plan wizard, plan
  inference, and all scoring changes — must be implemented as **explicit, deterministic
  code**: formulas, rules, lookups, precomputed tables, and structured user input.
- **The shipped feature must not call, depend on, or require any LLM, embedding model,
  or other AI/ML inference at runtime.** No parsing user text with an LLM, no semantic
  search, no probabilistic "AI recommendations." Every suggestion must be traceable to
  named constants, rules, and inputs.
- **Coding agents are expected to write this code** — that is the appropriate use of AI
  here. You are the implementation tool; the end product is ordinary application logic
  that runs without AI.
- When scoping or implementing any entry: prefer multiple-choice wizard answers, tag/
  archetype lookups, deck-analysis heuristics, and rule-based weight adjustments. If a
  proposal would add a runtime AI dependency, **stop and flag it to the user** — it is
  out of scope unless the user explicitly reopens it.
- This applies project-wide, not only to Entry 13. Entries 7–12 are formula/constant
  work. Entry 13 is a structured-intake + deterministic-plan algorithm. Do not substitute
  "use an LLM to figure out the deck's plan" for either.

## How an entry becomes a prompt
Don't draft a fix prompt until an entry reaches **Fix scoped**. Every prompt must state
that the deliverable is **deterministic algorithm code with no runtime AI** (see "Note
for coding agents"). A prompt built from an entry should include:
- Which file/function, with the current (verified, not assumed) behavior and line anchor
- The specific defect, described as one term/behavior on one side (Cuts or Adds)
- The desired behavior change — one term at a time
- Explicit "do not touch" list: other terms, the other side, and any relevant items from
  the known-quirks/intentional-design list
- A verification step: how to confirm the fix actually worked (e.g. "recompute score for
  test deck X, confirm curve bonus now factors in commander CMC")

When Status reaches **Prompt drafted**, add the copy-paste prompt to
`cuts-adds-ready-prompts.md` in **implementation order** (that file is the handoff queue for
partners). Keep detailed design notes in this backlog; keep only the runnable prompt in the
ready-prompts doc.

## Design interview methodology (for future planning sessions)

When working through an unscoped feature (such as Entry 13), do not jump straight to
proposing a design or implementation. Instead, use an iterative design interview with the
user.

The preferred workflow is:

1. Ask one focused design question at a time.
   - Keep questions small and independent.
   - Avoid bundling multiple unrelated decisions into one question.
   - When appropriate, present 3–5 clearly labeled options (A, B, C, D, etc.) that
     represent realistic design alternatives rather than leading the user toward one answer.
2. After the user answers:
   - Record exactly what they decided.
   - Do not reinterpret their answer into something broader without confirmation.
   - If their answer is ambiguous, ask a clarifying follow-up before proceeding.
3. Then explain your interpretation by separating it into two parts:
   - **Reasoning inferred:** What you believe motivated the user's decision. Clearly
     identify this as an inference, not a fact.
   - **Design implication:** What concrete behavior, architecture, UX, or scoring change
     should result from that decision.
4. Treat each answer as building on previous answers.
   - Do not revisit settled decisions unless they conflict with a newer one.
   - Use earlier decisions as constraints on future questions.
5. If you discover that a question has already been answered elsewhere (for example, by
   an existing backlog entry or previously agreed algorithm), do not ask the user to
   redesign it. Instead:
   - Reference the existing decision.
   - Continue from the next unresolved design question.
6. Separate conceptual design from implementation constraints.
   - First determine the ideal behavior.
   - Then separately discuss implementation feasibility (for example, whether something
     can be achieved without AI).
   - Do not let current implementation limitations redefine the intended design.
7. The objective of the interview is to progressively convert an abstract feature request
   into a collection of explicit, documented design decisions that can later be translated
   into implementation work.

This approach ensures that:

- every decision has documented reasoning,
- future coding agents understand why a decision was made,
- the user remains the source of truth for product behavior,
- and implementation prompts can be generated later without re-interviewing the user.

## Entry template
```
### [ID] Short title
- Status: Symptom noted / Needs investigation / Root cause confirmed / Fix scoped / Prompt drafted / Shipped
- Side: Cuts / Adds / Both
- Symptom: what bad or incorrect suggestion behavior was observed (ideally a concrete example deck/card)
- Suspected cause: hypothesis + file/line anchor if known — unverified until checked against actual code
- Confirmed cause: (filled in once investigated)
- Proposed fix: concrete, scoped to one term/formula piece
- Constraints: anything from known quirks / intentional design this fix must respect
- Open questions: anything to resolve with the user before drafting the fix prompt
```

---

## Backlog

### 1. Adds curve calc excludes commander CMC
- Status: **Prompt drafted** (2026-07-14) — **Runnable copy:** Prompt **3 of 3** in
  [`cuts-adds-ready-prompts.md`](./cuts-adds-ready-prompts.md). Prefer after Prompt 1
  (scoring rebalance); safe to parallel with Prompt 2 if curve-bucket edits don't collide.
- Side: Adds
- Symptom: Adds curve-gap bonus doesn't reflect the same curve Cuts sees, since Cuts includes
  commander CMC in curve buckets and Adds doesn't (decks.js:6460-6473 vs 6274)
- Confirmed cause: yes, per spec — verify still true at current line numbers before executing
- Proposed fix: include commander CMC bucket in Adds' curve calc, matching Cuts
- Constraints: only touch the curve-bucket construction in `_computeAddContext`; don't touch
  Cuts' curve logic, don't touch other Adds scoring terms
- Open questions: none

### 2. Plan-count token exclusion asymmetry
- Status: Needs investigation
- Side: Both (asymmetry between them)
- Symptom: none observed yet directly — flagged from spec, not from a concrete bad suggestion
- Suspected cause: Cuts' candidate pool excludes tokens; Adds' Plan-count filter doesn't (decks.js:6294 vs 6462)
- Confirmed cause: not yet — need to check whether this is actually causing visible bad
  suggestions before treating it as worth fixing
- Proposed fix: TBD, pending investigation and a decision on which behavior is "correct"
- Constraints: unclear whether this is intentional; do not assume it's a bug
- Open questions: does the user have a deck where token count is high enough that this
  would visibly change Plan-count math? Need a concrete case to justify prioritizing this.

### 3. `_deckSwapsEnabled(deck)` signature mismatch
- Status: Flag only — not currently scheduled as a fix
- Side: Both
- Symptom: none — no bad suggestion traced to this yet
- Suspected cause: function ignores its `deck` arg, toggle is user-wide not per-deck
- Confirmed cause: n/a
- Proposed fix: none planned; just needs to be known before any future planning-mode work
- Constraints: don't touch unless explicitly asked to make planning mode per-deck
- Open questions: none right now

### 4. Adds sends `tribes: []` to server
- Status: **Not a candidate for fixing** — intentional, partner's decision
- Side: Adds
- Symptom: n/a
- Constraints: do not draft a fix prompt for this without discussing with user first
- Open questions: none

### 5. Plan-only-deficit decks never fetch unowned cards
- Status: Needs investigation — **direction decided**; **bundled into Entry 13 v1** (Prompt
  drafted — see `cuts-adds-ready-prompts.md` Prompt 2). Implement with Entry 13; do not
  ship Entry 5 alone without plan schema.
- Side: Adds
- Symptom: likely explains "no suggestions" / "same suggestions forever" complaints —
  need a concrete example deck to confirm
- Suspected cause: unowned fetch requires a non-Plan deficit (decks.js:6679)
- Confirmed cause: not yet
- Proposed fix: **Direction decided by user: (a) — allow unowned fetch to trigger on
  Plan-only deficits too**, not just message the "no suggestions" state better. Blocked
  because "Plan" is currently defined purely negatively (no utility tag) — this says
  nothing about the deck's actual plan/theme, so backfilling against a Plan deficit
  without understanding the deck's plan would just fetch generic untagged cards.
- Constraints: do not implement the Plan-only backfill trigger until deck-plan
  identification is resolved. Don't conflate this with role-tag deficits — Plan is
  explicitly the absence of a role tag, so whatever identifies "the deck's plan" has to
  be a different signal than the existing ~36 utility role tags.
- Open questions:
  - **Plan identification:** largely resolved by Entry 13 interview (#20–30) — declared
    `winConditionId` + strategy IDs; analyze-first path for ≥80 cards. Entry 5 implements
    against Entry 13 v1 schema.
  - **Plan backfill filter/rank:** filter/rank by declared **strategy + win condition**
    (interview #30 v1 scope). Exact tag/heuristic mapping TBD in implementation prompt.
  - Still need a concrete example deck to confirm the "no suggestions" symptom in
    practice.

### 6. Owned/All Cards toggle for Adds
- Status: **Needs investigation** → design interview in progress (2026-07-14). Owned behavior
  already scoped; All Cards source locked by interview #1. Remaining: ranking, default,
  persistence, naming, Entry 5 interaction.
- Side: Adds
- Symptom: n/a — feature request. Recommendations today are owned-focused; user wants a
  second mode that opens the Adds pool to the local card DB (label TBD — not wedded to
  "All Cards").
- Proposed fix: Add a UI toggle on the Adds panel with two modes:
  - **Owned**: candidate pool = user's owned collection only — **no** server backfill,
    regardless of deficit state.
  - **Catalog / "All Cards" (name TBD)**: candidate pool = format + color-identity legal
    cards from the **local DB only** — independent of ownership. **No live Scryfall.**
- Constraints: Adds-only, do not touch Cuts. Do not touch quirk #4 (`tribes: []`,
  intentional). Do not touch the `CK_REQUIRED_ENABLERS` hard gate (15 qty-weighted
  enablers). This is a candidate-pool change, not a scoring-formula change — preserve
  existing scoring terms inside whichever mode is active. **Catalog mode sorts by score
  only** (interview #2) — no owned-first.
- Design interview (Entry 6):

| # | Question | Decision | Design implication |
|---|----------|----------|--------------------|
| 1 | What is "All Cards" mode's main job? | **Local DB catalog** — open recommendations beyond owned to **all cards in the local DB** (format/CI legal). User not particular about the control's label. Closes prior open question: **not** live Scryfall / not "every printed card." | Pool = local DB ∩ format + commander color identity. Keep "never live Scryfall." Label can be "All Cards," "Catalog," etc. — decide later or leave to implementer with a sensible default. |
| 2 | In catalog mode, does ownership affect ranking? | **A — Score only.** Ignore ownership; pure score order (top 8). | Catalog mode: disable owned-first sort / owned boost. Owned mode: ownership is the pool filter itself (no need for owned-first). |

- Open questions (remaining):
  - Default mode on first visit?
  - Toggle persistence (per deck / per user / session)?
  - How catalog mode interacts with Entry 5 plan-aware unowned/server backfill?
  - Final control label (low priority — user not wedded to a name).

### 7. Incorporate EDHREC rank into Cuts/Adds scoring
- Status: Needs investigation
- Side: Both
- Symptom: n/a — feature request. User wants a card's EDHREC popularity to factor into
  Cuts/Adds scoring, on the theory that overall play rate is a signal of how good a card is.
- Proposed fix: Add a new additive scoring term, E (EDHREC rank bonus), to the Adds
  formula: Score = (D × M) + C + E + V + T + K. **Weight-ordering decided:** E is
  weighted below D, M, and C/L (entry 11), but above V, T, and K. **Normalization
  decided:** E is derived from a normalized percentile of `edhrec_rank` within a
  filtered population (not raw rank), precomputed and cached server-side (matching the
  existing "never live Scryfall" pattern). Data source is Scryfall's `edhrec_rank` field
  — a plain integer on the standard card object (no bulk-data ingestion needed);
  confirmed live: starts at 1 and increases as popularity decreases, excludes basic
  lands, nullable for unranked cards. Sample values pulled directly from the Scryfall API
  (2026-07-10): Three Visits = 42, Explosive Vegetation = 1194.
- Constraints: `edhrec_rank` is a single global popularity number — not scoped to color
  identity, archetype, or role. Normalizing to a per-role-tag percentile blends it onto a
  comparable scale with the other additive terms without changing relative ordering
  within a shared pool. **EDHREC per-category endpoints are off the table** — ToS
  violation; do not use `json.edhrec.com` or scrape edhrec.com category pages (see
  archive entry 8).
  **E vs V interaction — decided.** V rewards role-tag breadth; E rewards popularity
  within the active role context — these should not fight each other. Multi-tag
  dampening (per-tag percentiles, optional dampening) applies **only inside E** — do not
  copy Cuts' subtractive multi-role discount onto Adds total score. When a candidate
  fills multiple active deficits, compute E once, using the percentile for the largest
  active deficit (do not sum per-tag E bonuses).
  **Canonical calibration case — Three Visits vs Growth Spiral (Simic, ramp + draw
  deficits, ramp larger).** E favors Three Visits (rank 42 → elite ramp percentile;
  Growth Spiral dampened as multi-tag) but is a tiebreaker-magnitude term, not enough
  alone to flip the ranking — entries 9, 10, and 11 close the rest of the gap. See entry
  10 for why D+V stacking currently wins anyway.
  **Verification target (once implemented):** Simic test deck with ramp + draw deficits;
  recompute scores; confirm Three Visits ranks above Growth Spiral; confirm E term alone
  favors Three Visits but does not override uncapped D+V stacking.
- Open questions:
  - **Precompute — decided.** Periodic job snapshots rank-within-population into the
    local DB; percentiles never computed live/per-suggestion.
  - **Minimum population size — decided.** Hard floor of **8** ranked cards. Below 8, no
    percentile computed; E falls back to neutral (no bonus/penalty). (Independent
    constant from `CK_REQUIRED_ENABLERS` = 15.)
  - **Multi-tag cards — decided.** Precompute stores **one percentile per role tag**
    (e.g. `ramp_percentile` and `card_draw_percentile` separately), each against that
    tag's own population (subject to the floor above). At scoring time, the role deficit
    being filled selects which stored percentile is used. This sidesteps entry 8's
    unsolved "primary role" disambiguation entirely.
  - **EDHREC vs USD price distortion — decided.** High-dollar cards are played less than
    their power level warrants, depressing rank. E should be price-adjusted (dampen rank
    penalty proportional to USD above a baseline, capped) so expensive staples aren't
    systematically underrated. Implementing agent must document chosen formula; Three
    Visits (rank ~42) must remain elite after adjustment.
  - **Percentile-to-E-bonus conversion — decided.** Smooth curve, not discrete tiers (e.g.
    `E = K_E × percentile^n`, price adjustment layered on top). Avoids cliff-edge
    artifacts at tier boundaries. Implementing agent picks the function form and
    documents it.
  - **E on the Cuts side — decided.** E extends to Cuts as well as Adds. Mechanics
    (which Cuts term it modifies, sign/weight) still TBD — needs its own scoping pass.
  - **Scoring rebalance with entries 9, 10, 11** — should ship as part of the same design
    pass before or with E, so terms don't fight or double-count. Entry 12 may join once
    spellslinger detection is confirmed in repo.

### 9. Adds mana pip color restrictiveness (castability)
- Status: Symptom noted
- Side: Adds
- Symptom: predicted bad suggestion — flexible/multi-pip cards ranked too highly vs
  easier-to-cast options at the same CMC. User example: `{G}{U}` Growth Spiral (2 CMC,
  two colored pips) treated as comparably easy to cast as `{1}{G}` Three Visits (2 CMC,
  one colored pip + one generic). Fewer colored pips = more castable, distinct from
  role-tag versatility (entry 10).
- Suspected cause: Adds scoring has no term that distinguishes generic-heavy mana costs
  from multi-colored-pip costs. CMC is factored into curve-gap bonus (C) but pip
  restrictiveness is not. Verify in `_scoreAddCandidate` / mana-cost parsing at current
  line numbers.
- Confirmed cause: not yet — need to verify Adds scoring ignores colored-pip count.
- Proposed fix: add a new subtractive term **P** (pip restrictiveness penalty) as its
  **own scoring term — decided, not folded into C.** Derive from the card's mana cost:
  - Colored pips (W/U/B/R/G): full weight — count toward P.
  - Generic `{1}`/`{2}`/etc.: do not add restrictiveness.
  - **`{C}` (colorless) — decided: does not count toward P.**
  - **Hybrid pips (e.g. `{G/U}`) — decided: dampened, roughly half weight** of a fixed
    colored pip.
  - **Phyrexian pips (e.g. `{G/P}`) — decided: negative contribution to P (a small
    bonus, not neutral)** — payable with life, more flexible than even colorless.
    Exact magnitude TBD, calibrate alongside other P constants.
  - **Variable costs (`{X}`) — decided: 0 contribution to P.**
  **User intent:** Growth Spiral should take a meaningful ding vs Three Visits partly
  because of its two colored pips. Exact formula/weight TBD — must sit below D, M, C/L,
  and E in influence but be strong enough to matter in same-CMC comparisons. Coordinate
  with entry 11 in the same design pass.
- Constraints: Adds-only unless user later asks for Cuts symmetry. Do not conflate with
  color-identity legality (already a candidate-pool filter). Distinct from role-tag
  versatility (V). Scryfall mana-cost data is already local. Do not touch CK hard gate,
  tribes, or candidate-pool logic.
- Open questions:
  - **User framing (decided):** penalize colored pips regardless of on-color status,
    even within the commander's own color identity.
  - Numeric weight/cap for P, and the exact hybrid dampening factor (~50% as a starting
    point) — needs calibration alongside entries 10 and 11 rebalance.
  - **Cross-cutting: effective CMC for `{X}` spells — decided, applies beyond P.** For
    CMC-based terms (curve bonus C, entry 11's L), treat `{X}` as **X = 3**. Does not
    affect P itself. Implementing agent should apply the X=3 convention consistently
    wherever CMC-based scoring terms read a card's CMC.

### 10. Versatility overweight — flexible cards beat dedicated staples (Growth Spiral vs Three Visits)
- Status: Symptom noted
- Side: Adds
- Symptom: predicted bad suggestion — when a deck has both Ramp and Card Draw deficits,
  Growth Spiral likely outscores Three Visits because it fills two deficits (D), earns
  versatility bonus (V), and has strong EDHREC rank (E, once entry 7 ships) — even
  though Three Visits is among the best ramp cards in the format and should rank above
  Growth Spiral when ramp is the larger active deficit. User position: **versatility is
  valuable but is NOT everything**; dedicated S-tier role cards should beat flexible
  mid-tier dual-role cards when the **largest deficit** favors single-role excellence.
- Suspected cause: compound stacking — D scales with deficits filled (double-counts
  multi-role candidates when multiple deficits are active), V adds a separate breadth
  bonus. No pip restrictiveness term (entry 9). No CMC efficiency term (entry 11).
  Verify actual V formula and D multi-deficit scaling in `_scoreAddCandidate`.
- Confirmed cause: not yet — need to verify current V weight and whether D multiplies or
  sums per deficit.
- Proposed fix: **Primary lever decided — sublinear D scaling (Option A).** Rebalance
  across existing terms, not deleting V:
  - **D sublinear scaling (PRIMARY):** when a candidate matches multiple active deficits,
    sort matched deficit magnitudes descending; apply weights `1.0, ~0.40, ~0.20` for
    1st/2nd/3rd deficit credit. Single-deficit candidates unchanged.
  - **V dampening (tertiary):** keep V positive; dampen inside V for 2+ utility tags
    (~50% contribution from 2nd tag onward, or similar). **Do not** add Cuts-style
    subtractive multi-role discount to Adds total score.
  - **Combine with entry 9** (P), **entry 11** (L), and **entry 7** (E, price-aware,
    dampened for multi-tag inside E).
  - **Acceptance bar (decided):** Three Visits must rank **above** Growth Spiral when
    **ramp is the larger deficit** — not merely >50% of mixed-deficit cases.
- Constraints: Versatility should remain a positive signal on Adds (unlike Cuts'
  penalty) — flexible cards that legitimately solve two problems should still rank well
  when both deficits are severe and comparable. Goal is to stop flexibility from
  systematically beating dedicated best-in-slot cards when the largest gap is
  role-specific excellence. Coordinate with entries 7, 9, and 11 so terms don't
  double-penalize or double-reward the same property.
  **Canonical test case:** see entry 7's Three Visits vs Growth Spiral calibration case.
- Open questions:
  - Exact sublinear weights and V dampening constants — calibrate on canonical test case
    in repo with logged term breakdown.
  - Price-aware E (entry 7) ships in same pass — decided yes.

### 11. CMC efficiency — ignore curve bonus (C), reward lower CMC (L) for interaction roles
- Status: **Needs investigation** — algorithm design decided; **project role-tag IDs must
  be mapped in repo** before prompt can be drafted (this project uses different tag names/IDs
  than Scryfall `otag:` slugs; do not assume 1:1 mapping from the reference list below).
- Side: Adds
- Symptom: predicted bad suggestion — curve-gap bonus (C) treats higher-CMC spells as
  more valuable when they fill a higher curve slot, even when lower-CMC spells are
  strictly better in the same role. User examples:
  - **Three Visits vs Cultivate:** both ramp; Cultivate (3 CMC) must not outrank Three
    Visits (2 CMC) because C rewards filling the 3-drop slot.
  - Same pattern expected for cheap removal, protection, combat tricks, and pump.
- Suspected cause: Adds scoring applies curve-gap bonus (C) uniformly to all
  role-tagged candidates, with no inverse "cheaper is better" signal for roles where
  mana efficiency dominates. Verify in `_scoreAddCandidate` and curve-bucket logic at
  current line numbers.
- Confirmed cause: not yet in repo — confirmed at spec level that documented formula has
  no L term and no role-specific C suppression.
- Proposed fix: add new additive term **L** (CMC efficiency bonus) and **suppress C** for
  candidates in efficiency-mode roles. Coordinate with entries 7, 9, 10, and 12 in the
  same design pass.
  **Weight order (decided):** `D, M` > `C or L` > `E` > `B` > `P` > `V` > `T, K` (B =
  entry 12 creature body bonus).
  **Behavior:**
  - If candidate has ≥1 **efficiency-mode project role tag** AND is **not a land**: set
    **`C = 0`**, compute **`L = K_L × max(0, CMC_REF − CMC)`** with starting constants
    `CMC_REF = 4`, `K_L` tuned in repo.
  - Otherwise: existing C unchanged, **`L = 0`**.
  - Lands are excluded from L even if tagged ramp (avoids breaking `land-ramp`).
  **Implementation prerequisite — project tag mapping (REPO REQUIRED):** the
  implementing agent must locate the project's internal role-tag constants/enums and
  build `EFFICIENCY_MODE_PROJECT_TAGS` from the semantic categories below. Scryfall
  `otag:` slugs are **reference only** — map each category to whatever IDs/names the
  codebase actually uses.
  ---
  **Tier 1 — efficiency mode (required; user-confirmed roles):**
  | Semantic role | Scryfall `otag:` reference slugs |
  |---------------|----------------------------------------------------------------|
  | **Ramp** | `ramp` |
  | **Removal** | `removal`, `spot-removal`, all `removal-*` subtags |
  | **Protection** | `protection`, `damage-prevention*`, `gives-protection`, `gains-protection`, `gives-hexproof`, `gains-hexproof` |
  | **Combat trick** | `combat-trick` |
  | **Pump** | `giant-growth`, `giant-growth-with-set-mechanic` |
  **Tier 2 — efficiency mode (recommended v1):**
  | Semantic role | Scryfall `otag:` reference slugs |
  |---------------|----------------------------------|
  | **Counterspell** | `counterspell` + subtags |
  | **Direct damage / burn removal** | `burn`, `burn-any`, `burn-creature`, `burn-planeswalker`, `burn-player`, `burn-player-each` |
  | **Bounce / unsummon** | `bounce`, `bounce-self` |
  | **Combat fog** | `fog`, `fog-selective`, `pseudo-fog` |
  | **Silence / hard no** | `silence`, `prevent-cast` |
  | **Hand disruption** | `hand-disruption`, `discard` |
  **Tier 3 — discuss before including:**
  | Semantic role | Scryfall `otag:` reference slugs | Caveat |
  |---------------|----------------------------------|--------|
  | **Tutor** | `tutor` | Cheap tutors dominate; some 4–5 CMC tutors still staples |
  | **Recursion / reanimation** | `recursion`, `reanimate` | Cheap often king; haymakers break pattern |
  | **Cantrip draw** | `cantrip`, `pure-draw`, `impulsive-draw` | Not same as spot-interaction efficiency |
  | **Fight/bite tricks** | `fight`, `one-sided-fight`, `bite` | Overlap with combat-trick |
  **Explicit exclusions — keep normal C, do NOT apply L:**
  | Semantic role | Scryfall `otag:` reference slugs | Why |
  |---------------|----------------------------------|-----|
  | **Board wipe** | `sweeper`, `sweeper-one-sided`, `sweeper-graveyard`, `board-reset`, `multi-removal` | Wipes belong at 4–6 CMC |
  | **Draw engines** | `draw-engine`, `repeatable-draw`, `repeatable-card-advantage` | Rhystic Study at 3 beats cantrips in role quality |
  | **Card Draw (general)** | `card-advantage`, `draw` | Mix of cantrips and engines |
  | **Land-based ramp** | `land-ramp`, `multi-land-ramp`, `bounceland` | Lands use different CMC economics |
  | **Plan / untagged** | *(no utility tag)* | Filler and haymakers — curve still matters |
  | **Anthems / finishers** | `anthem`, `group-slug` | Mid–high CMC payoffs |
  ---
  **Updated Adds formula (when entries 7/9/10/11/12 ship together):**
  `Score = (D × M) + C_eff + L + E + B − P + V + T + K`
  where `C_eff = 0` when L applies, else existing C. See entry 12 for B.
  **Verification targets:**
  1. Ramp deficit active: Three Visits ranks above Cultivate (L drives this).
  2. Ramp deficit > draw deficit: Three Visits ranks above Growth Spiral (primarily
     entry 10's D sublinear; L/P/E must not break this).
  3. Board-wipe deficit only: sweeper candidates still receive normal C.
  4. Log term breakdown: D, M, C, L, E, P, V, T, K for TV vs Cultivate and TV vs GS.
- Constraints: Adds-only. Do not change Cuts curve logic. Do not use Scryfall `otag:`
  slugs directly unless the codebase already keys off them. Do not apply L to lands.
  Coordinate constants with entries 9 and 10. Entry 12 interacts with ramp comparisons —
  calibrate L and B together on STE vs Rampant Growth and Wood Elves vs Rampant Growth.
  Do not touch CK gate, tribes, candidate pool.
- Open questions:
  - **Project tag mapping** — blocked on repo access.
  - **Tier 2 tags in v1 — decided: yes.**
  - `K_L` and `CMC_REF` calibration vs entry 10/12 constants.
  - Tier 3 tags — include in v1?
  - **Effective CMC for `{X}` spells — decided (see entry 9).** X = 3 for CMC-based
    terms, including L.

### 12. Creature body bonus — creatures beat non-creature spells (non-spellslinger)
- Status: Symptom noted
- Side: Adds
- Symptom: predicted bad suggestion — for ramp (and likely other efficiency roles),
  non-creature spells with higher EDHREC popularity outrank strictly better creature
  versions. User examples:
  - **Sakura-Tribe Elder vs Rampant Growth:** STE is significantly better; Rampant
    Growth has higher EDHREC score but should not win in a typical creature-based deck.
  - **Wood Elves vs Rampant Growth:** Wood Elves (3 CMC creature, ETB ramp) is probably
    better than Rampant Growth (2 CMC sorcery) — "Three Visits on a creature" — even
    though entry 11's L term favors lower CMC on spells.
- Suspected cause: Adds scoring values the spell effect but not the creature body
  (blocking, chump blocking, sacrifice/fodder synergy, attack pressure, death triggers).
  E (entry 7) may further favor popular sorceries over creatures. No archetype-aware
  creature premium exists. Verify card-type handling in `_scoreAddCandidate`.
- Confirmed cause: not yet — need repo to confirm no creature-type bonus and to locate
  spellslinger/archetype detection.
- Proposed fix: add new additive term **B** (body bonus), gated by deck archetype.
  - **Gate:** apply B only when `!deckIsSpellslinger(deck)`.
  - **Candidate:** a Creature AND matches an active deficit via a utility role tag.
  - **v1 scope:** full B weight when filling **Ramp** deficit; optional extension to
    Removal/Protection on-a-stick creatures in v2.
  - **Formula sketch:** `B = K_B` base, with `K_B_RAMP` ≥ `K_B` when largest active
    deficit is Ramp.
  - **Weight order:** B sits below D, M, C/L, and E but above P — must be large enough
    to flip STE over Rampant Growth when E favors the sorcery, and large enough that
    Wood Elves beats Rampant Growth despite L's 1-point CMC edge to RG.
  - **Spellslinger detection (REPO REQUIRED):** use existing archetype detection if
    present; fallback heuristic TBD in repo (document chosen signal). When spellslinger:
    B = 0.
  - **Interaction with entry 11:** creatures in efficiency-mode roles still get L, but
    also get B when not spellslinger. Consider an effective-ramp-CMC for ETB-ramp
    creatures (e.g. Wood Elves as CMC 2 for L) only if B alone can't flip WE vs RG.
  **Verification targets:**
  1. Non-spellslinger green ramp deck: Sakura-Tribe Elder ranks above Rampant Growth.
  2. Same deck: Wood Elves ranks above Rampant Growth.
  3. Spellslinger deck: Rampant Growth can rank above STE — B = 0.
  4. Log term breakdown including B for STE vs RG and WE vs RG.
- Constraints: Adds-only. B is additive only. Do not apply B to lands, tokens, or
  non-creature types. Coordinate calibration with entries 7 and 11. Do not touch CK
  gate, tribes, candidate pool. Spellslinger detection must respect existing archetype
  override UX if present.
- Open questions:
  - Exact spellslinger signal in repo — archetype enum name, slider, or heuristic?
  - Extend B beyond Ramp in v1 (removal creatures, protection creatures)?
  - `K_B` vs `K_B_RAMP` numeric values — calibrate on STE/RG and WE/RG with logged terms.
  - ETB-ramp effective-CMC adjustment inside L — needed or does B alone suffice?

### 13. User-declared deck plan (guided wizard intake)
- Status: **Prompt drafted** (2026-07-13) — v1 ready for partner implementation. **Runnable
  prompt:** Prompt 2 in [`cuts-adds-ready-prompts.md`](./cuts-adds-ready-prompts.md) (also
  [`entry-13-v1-implementation-prompt.md`](./entry-13-v1-implementation-prompt.md)). Run in
  main app repo after Prompt 1 when possible. Do not execute until user says start.
  Supersedes the original "natural language + LLM parse" direction as the primary
  implementation path (see Design decisions below).
- Side: Adds (primary); Cuts plan-awareness deferred to v2 (interview #29)
- Symptom: n/a — feature request. Currently "Plan" has no positive definition; the app
  has no way for the user to directly state a deck's actual gameplan/win condition. This
  blocks meaningful Plan-aware backfill (entry 5) and Plan-aware Cuts scoring.
- Confirmed cause: n/a — design-level only so far
- Proposed fix: Add a **guided wizard** (primary intake) that collects structured plan
  information the recommendation engine consumes via a **deterministic algorithm — no AI at
  runtime, ever** (see "Note for coding agents" at top of doc). The wizard is itself part
  of the algorithm: multiple-choice answers, deck-analysis heuristics, and explicit
  user corrections — not LLM interpretation. Output must be structured enough for scoring
  to consume — not just stored as a text blob. Must be compatible with entry 5's
  downstream plan interface so declared and inferred plan can feed scoring the same way
  once entry 5 unblocks.
  **Overall goal:** help the engine understand what the player is trying to accomplish so
  recommendations align with the deck's intended identity, not just generic role deficits.
  **Wizard behavior (high level):**
  - Adaptive by experience level (Beginner / Intermediate / Advanced) — early question
    sets wording depth and pacing.
  - If insufficient cards exist: skip deck analysis, begin with user questions.
  - If sufficient cards exist: analyze deck first → generate observations → ask questions
    informed by those observations (use existing info before asking user to repeat it).
  - Collect Primary / Secondary / Tertiary plan (and more if desired); encourage 1–2
    plans but no hard limit.
  - Questions primarily multiple-choice, each with a "Show More Options" button; all
    questions skippable.
  - User can correct incorrect analysis conclusions — user input is authoritative.
  - User can **go back and edit any prior answer at any point** in the wizard (back button,
    arrow, or equivalent) — plan intake is never write-once.
  - Wizard determines when it has enough info, then offers: "I have what I need to make
    recommendations, but answering more questions can make them stronger." User chooses
    whether to continue; wizard does not auto-stop.
  - Works for new deck creation **and** optimizing existing decks.
  **How plan influences recommendations (conceptual):**
  - Functional roles (Ramp, Draw, Removal, Protection, etc.) are foundational — the
    "vegetables." Plan is also a role, but represents identity/strategy/what makes the deck
    unique — the "exciting" part. Tags and Plan are separate: tags provide signals; Plan is
    a higher-level concept built from deck analysis + existing tags + user intent. Plan does
    not replace tagging.
  - **"Eat your veggies first":** recommendations prioritize (1) functional role
    deficiencies, then (2) plan enhancements. Functional priorities use baseline
    deck-building principles, adjusted by Plan and user preferences.
  - Ideal: Plan influences weighting of other roles (e.g. sacrifice deck values cards
    differently than control). **v1 ships equal-weight Plan role only**; hybrid modifiers
    are **v2** (interview #30). See **v1 / v2 scope** under design interview decisions.
  - Recommendations organized by priority; when a card satisfies multiple roles, UI should
    explain those roles.
  - **Do not redesign** the multi-role scoring algorithm (entries 7, 9, 10, 11, 12) —
    Entry 13 provides better plan information for that existing algorithm to consume.
- Constraints:
  - **No runtime AI — hard constraint.** The shipped feature must not call, depend on, or
    require any LLM or other AI/ML inference. Wizard intake, deck analysis, plan
    inference, and scoring integration must all be deterministic code. Coding agents write
    the code; the product does not use AI.
  - Wizard produces structured data directly (multiple choice, skippable questions, user
    corrections) — never free-text → LLM parse.
  - User can navigate **back** to edit any prior wizard answer at any time (back button,
    arrow, or equivalent); intake is never write-once.
  - Should complement, not replace, archetype + deck-analysis inference for **pre-wizard
    inference** — declared plan takes precedence over inferred plan when present; user
    corrections are authoritative. **Post-wizard:** declared plan **overrides** archetype
    for recipe/scoring (interview #28).
  - Do not implement until the plan output schema is settled enough for entry 5 and
    scoring to consume through a shared downstream interface.
  - Entry 13 does not redesign entries 7/9/10/11/12 scoring terms.
- Open questions:
  - **v1 catalog + algorithms** — settled in **v1 algorithm spec** below (user approved
    option A, 2026-07-13). Constants may be tuned during implementation; IDs are stable.
  - **Repo anchors** — TBD at implementation Step 0 in main app repo (not in this archive).
  - Optional free-text notes — v2.

#### Design decisions (2026-07-13 chat — design reference, not implementation prompt)
Captured from a design session (ChatGPT; some context was misunderstood, but decisions
below are retained as valuable). Supersedes the original LLM-first proposal in this entry.
**Agent clarification (2026-07-13):** we are building an algorithm. Coding agents may use
AI to write the code; the shipped product must not use AI at runtime.

| # | Decision |
|---|----------|
| 1 | **Wizard first** — guided wizard is primary intake; only structured answers affect scoring |
| 2 | **Multiple choice primary** — each question has "Show More Options"; no AI needed to interpret arbitrary input |
| 3 | **Adaptive wizard** — early question sets Beginner / Intermediate / Advanced; wording and pacing adapt |
| 4 | **Deck-creation vs existing-deck paths** — insufficient cards → questions first; sufficient cards → analyze deck first, then ask informed questions |
| 5 | **When analysis happens** — analyze → observations → informed questions (or skip analysis if too few cards) |
| 6 | **Primary / Secondary / Tertiary plan** — collect explicitly; encourage 1–2 plans, no hard limit |
| 7 | **All questions skippable** — wizard never requires every question answered |
| 8 | **Wizard completion** — system signals when it has enough info; user chooses to continue or stop (no auto-stop) |
| 9 | **User can correct system** — if analysis is wrong, user overrides; user input is authoritative |
| 10 | **Existing deck optimization** — wizard helps optimize existing decks, not only new-deck creation |
| 11 | **Plan ≠ tags** — tags are signals; Plan is higher-level (analysis + tags + user intent); tagging system stays intact |
| 12 | **Plan vs roles clarified** — functional roles are foundational "vegetables"; Plan is the exciting identity/strategy role; don't forget either |
| 13 | **Plan influences role weighting** — ideal: Plan adjusts how other roles are valued; v1 fallback: Plan as equal-weight role if dynamic weighting too complex |
| 14 | **"Eat your veggies first"** — functional role deficiencies before plan enhancements |
| 15 | **Functional priorities** — baseline principles, adjusted by Plan and user preferences; players may intentionally deviate |
| 16 | **Recommendation presentation** — organized by priority; multi-role cards explained in UI |
| 17 | **Don't redesign scoring algorithm** — entries 7/9/10/11/12 already govern multi-role weighting; Entry 13 feeds them better plan data |
| 18 | **Rethink original Entry 13** — natural language + LLM parse is rejected; wizard is the structured data source |
| 19 | **Algorithm, not AI product** — deterministic rules/formulas end-to-end; coding agents write the code, but runtime must not use LLM or other AI inference |

#### Design interview decisions (2026-07-13 — user-confirmed)
Iterative design interview per "Design interview methodology" above. Each row is one
user-confirmed answer; do not reinterpret without confirmation.

| # | Question | Answer | Design implication |
|---|----------|--------|-------------------|
| 20 | When the wizard asks for **Primary plan**, what kind of thing is the user identifying? | **Revised (2026-07-13): Strategy/archetype AND win condition** — plan dictates how the deck will win. Supersedes initial answer A (strategy only). User cited Marshall Sutcliffe, ["My Most Important Deck-Building Rule"](https://magic.wizards.com/en/news/feature/my-most-important-deck-building-rule-2018-02-08): every deck needs a mission statement combining strategy and win method. | Schema and wizard must capture both dimensions of "plan." Exact intake shape TBD — see interview question #21 (unified mission statement vs separate picks). Plan backfill/scoring should use both strategy identity and win method when ranking/filtering. |
| 21 | How should the wizard capture strategy + win condition? | **B — Two separate questions** — pick strategy/archetype first, then win condition as a distinct follow-up; both together constitute the plan for that slot. | Wizard uses two question types: strategy/archetype picker and win-condition picker. Schema stores `winConditionId` (deck-wide) plus `strategyId` per Primary/Secondary/Tertiary slot. See #22 for how win condition relates to plan slots. |
| 22 | With Primary/Secondary/Tertiary plan slots, how does win condition apply? | **A — Once per deck** — win condition asked once for the whole deck; Primary/Secondary/Tertiary are strategy/archetype only. | Single `winConditionId` on the deck plan object. Plan slots hold strategy IDs only. Scoring/backfill: one win method gates all strategies; hybrid decks express multiple strategies under one win condition. |
| 23 | When in the wizard should the win-condition question appear? | **Conditional — D then A:** If sufficient deck data exists → analyze deck first, surface observations, then ask win condition with suggestions informed by analysis (user can correct). If insufficient data → ask win condition **before any strategy** (mission-statement-first, option A). | Two wizard paths for win-condition placement. Analyze-first path pre-suggests win condition from deck heuristics; questions-first path leads with win condition. Both paths then proceed to strategy/archetype slots. "Sufficient cards" threshold — see interview question #24. |
| 24 | At how many cards does the analyze-first path kick in? | **D — 80 cards** | Decks with ≥80 cards: analyze → observations → informed win-condition question. Decks with &lt;80 cards: win condition before any strategy. Threshold is a named constant in wizard logic. |
| 25 | On the ≥80-card analyze-first path, what to show before asking win condition? | **D — Correctable observation chips** — each inference (win condition, strategies, archetype, etc.) shown as a chip the user confirms, corrects, or skips before proceeding to formal questions. | Analyze-first UX: deterministic inferences rendered as chips; user input authoritative per design decision #9. Chip set likely includes win condition, strategy/archetype, and existing archetype detection output. Formal question phase behavior relative to chips — see interview question #26. |
| 26 | After observation chips, what happens in the formal question phase? | **B + pre-fill on skipped** — confirmed or corrected chip **skips** the corresponding formal question; skipped or missing chip **runs** the formal question, **pre-filled** from inference when available. **UX guardrail:** chip **Correct** opens the **same** multiple-choice picker as the formal question (including "Show More Options"). **Always editable:** user can go back and change any prior answer at any point (back button, arrow, or equivalent). | Per-field intake state: `chip-confirmed`, `chip-corrected`, `formal`, or `skipped→formal`. Chip correction and formal questions share one picker component/catalog. Back navigation reopens any field without losing later progress. |
| 27 | Ideally, how should declared plan influence functional role priorities? | **C — Hybrid** — plan nudges role weights modestly; "eat your veggies first" ordering always preserved (functional deficits always outrank plan enhancements). | Conceptual scoring model: plan-derived modifiers adjust recipe/threshold weights within bounds; deficit priority queue unchanged. v1 implementation may still use equal-weight Plan fallback if hybrid modifiers are too complex — see separate implementation question when ready. Confirms design decision #13 ideal in user interview. |
| 28 | How should wizard-declared plan interact with existing archetype detection? | **A — Override** — declared plan replaces archetype for recipe/scoring adjustments. | When user completes wizard plan intake, declared plan (win condition + strategies) supersedes detected/overridden archetype for recipe threshold and scoring weight adjustments. Archetype detection may still seed inference chips on analyze-first path, but declared plan wins. Refines entry constraint "complement archetype" for post-wizard runtime behavior. |
| 29 | Should Cuts use declared plan in v1? | **C — Cuts deferred to v2** — schema supports plan-aware Cuts later; no Cuts scoring changes in v1. | v1 scope: Adds + Plan backfill (entry 5) + wizard intake only. Plan schema should include fields/hooks for future Cuts shielding (e.g. strategy-aligned cards). Entry 13 side remains Adds-primary for v1 implementation. |
| 30 | v1 scoring integration — how does declared plan affect Adds first? | **D — Phased** (agent recommendation accepted): **v1 = B** (equal-weight Plan role + plan-aware Plan backfill); **v2 = A** (hybrid bounded role-weight modifiers per #27). See **v1 / v2 scope** below. | **v1 ships:** wizard + schema; equal-weight Plan deficit in Adds; plan-aware Plan backfill (unblocks entry 5); declared plan overrides archetype for plan-related recipe/backfill paths; schema hooks for v2. **v2 required:** hybrid role-weight nudges (#27), Cuts plan-awareness (#29), and any deferred catalog depth. v1 alone is **not** the final scoring model. |
| 31 | v1 strategy/archetype catalog scope? | **C + dynamic caveat** — minimal **~5–6 primary** options (not a fixed global list); full catalog via "Show More Options." **≥80 cards:** top 5–6 ranked by **deck analysis**. **&lt;80 cards:** user confirms **commander** first (if not already set); top 5–6 ranked by **commander affinity** lookup. **Win condition** picker gets the **same dynamic treatment** (analysis or commander). User confirmed 2026-07-13. | v1 wizard needs: `rankStrategiesForDeck`, `rankStrategiesForCommander`, `rankWinConditionsForDeck`, `rankWinConditionsForCommander`; commander confirmation step on &lt;80 path before plan questions; static fallback primary list when confidence low. Feasibility for v1 — see interview note below #31 in **v1 / v2 scope**. |

#### v1 / v2 scope (Entry 13 — user-confirmed via interview #29–31; feasibility trim 2026-07-13)

**Context:** `main` has Entry 13 design decisions #1–19 (wizard concept, no runtime AI).
PR #2 adds interview decisions #20–31 and design-interview methodology. v1 below is the
**feasible MVP** — enough to unblock entry 5 and improve Adds; v2 completes the vision.

##### v1 MVP — ship this (one implementation prompt)

**A. Plan schema & persistence**
- `winConditionId` (deck-wide, required for wizard completion)
- `primaryStrategyId` (required for wizard completion)
- `secondaryStrategyId` (optional, skippable)
- v2 hooks on schema for: `tertiaryStrategyId`, hybrid modifiers, Cuts shielding
- Per-field `source` metadata (`chip-*`, `formal`, etc.) for debugging

**B. Wizard — core flow (both paths)**
- Multiple-choice pickers + "Show More Options" + all questions skippable (#7)
- Back navigation to edit any prior answer (#26)
- Two question types: win condition, then strategy (#21–22) — win condition once per deck
- **&lt;80 cards:** confirm commander if missing → mission-statement-first (#23–24)
- **≥80 cards:** lightweight deck analysis → **simplified observations** (see C below)
- **Completion:** wizard done when win condition + primary strategy are set (or user exits
  early); optional secondary strategy question offered once — no tertiary in v1

**C. Analyze-first path (≥80 cards) — v1 simplified chips**
- Run deterministic deck analysis (tag clustering / role-tag ratios — reuse existing tags)
- Show **at most 3 observation chips:** inferred win condition, inferred primary strategy,
  optional archetype hint
- Chip behavior per #26: confirm/correct skips formal Q; skip → formal Q pre-filled;
  Correct uses shared picker
- **Defer to v2:** chips for secondary/tertiary strategy; rich multi-signal observation panel

**D. Dynamic picker ordering (#31) — v1 pragmatic**
- Top ~5–6 options ranked before "Show More Options" for **both** strategy and win condition
- **≥80 cards:** rank from deck-analysis signals (same engine as chips)
- **&lt;80 cards:** rank from **commander oracle keyword rules** (sacrifice, token, cast, etc.)
- Full catalog: **~15 strategies + ~8 win conditions** in v1 (not ~30); expand in v2
- **Static fallback** primary list when confidence low (always show something sensible)
- **Defer to v2:** curated explicit commander→affinity table for top 200+ commanders;
  win-condition inference accuracy tuning

**E. Adds integration — the v1 payoff**
- **Equal-weight Plan role** in Adds scoring (#30) — no hybrid role nudges
- **Plan-aware backfill** (unblocks entry 5): allow unowned fetch on Plan-only deficits;
  filter/rank by declared `winConditionId` + `primaryStrategyId` (+ `secondaryStrategyId`
  when set)
- Declared plan **overrides archetype** for plan-related backfill/filter paths only (#28) —
  do not rewrite full recipe engine in v1

**F. Explicitly out of v1**
- Hybrid role-weight modifiers (#27)
- Cuts plan-awareness (#29)
- Experience-level branching (#3 — Beginner/Intermediate/Advanced): **single wording in v1**
- Tertiary strategy slot
- Optional free-text plan notes
- Multi-role recommendation explanations in UI (#16)
- Full "answer more for stronger recommendations" continue loop (#8) — v1: simple Done +
  optional secondary strategy step

##### v2 — required follow-up (documented, not optional polish)

| Area | v2 work |
|------|---------|
| **Scoring** | Hybrid bounded role-weight modifiers (#27); veggies-first preserved |
| **Cuts** | Plan-aware shielding for strategy-aligned cards (#29) |
| **Wizard depth** | Tertiary strategy; experience-level branching; richer completion/continue UX (#8) |
| **Analysis UX** | More observation chips; secondary/tertiary inference; calibration |
| **Catalogs** | Expand to ~30+ strategies; commander explicit-affinity table; ranking tuning |
| **Presentation** | Multi-role card explanations (#16); optional display-only notes |

##### Suggested implementation order (v1)

1. Schema + persistence
2. Minimal wizard (&lt;80 path first — commander → win con → primary strategy)
3. Plan-aware backfill (entry 5) — **validate payoff early**
4. ≥80 analysis + simplified chips + dynamic picker ordering
5. Secondary strategy (optional step) + archetype override for backfill

**v1 deliverables (Fix scoped target):** See **v1 MVP** bullets A–F above (feasibility
trim 2026-07-13). Algorithm detail in **v1 algorithm spec** below.

#### v1 algorithm spec (Fix scoped — agent draft approved 2026-07-13)

##### Named constants

| Constant | Value | Notes |
|----------|-------|-------|
| `PLAN_WIZARD_ANALYZE_THRESHOLD` | `80` | Interview #24 |
| `PLAN_PRIMARY_OPTIONS_COUNT` | `6` | Top options before "Show More Options" |
| `PLAN_INFERENCE_CONFIDENCE_MIN` | `0.35` | Below → static fallback / no chip pre-fill |
| `PLAN_CHIP_MAX` | `3` | Win con, primary strategy, archetype hint |
| `PLAN_TAG_SIGNAL_WEIGHT` | `1.0` | Per matching deck tag signal |
| `PLAN_ORACLE_SIGNAL_WEIGHT` | `0.5` | Per commander oracle keyword hit (cap 3 hits/category) |

##### v1 strategy catalog (15 — stable IDs)

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

**Static fallback primary (6)** when confidence low: `strategy.tokens`, `strategy.sacrifice`,
`strategy.spellslinger`, `strategy.tribal`, `strategy.control`, `strategy.other`

##### v1 win-condition catalog (8 — stable IDs)

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

**Static fallback primary (5)** when confidence low: `wincon.combat`, `wincon.commander_damage`,
`wincon.combo`, `wincon.life_drain`, `wincon.value`

##### Commander oracle keyword rules (`rankForCommander`)

Deterministic substring match on commander oracle text (case-insensitive). Each hit adds
`PLAN_ORACLE_SIGNAL_WEIGHT` to listed strategy or win-condition IDs (cap 3 hits per ID).

**Strategy signals (examples):**

| Keyword / phrase | Boosts |
|------------------|--------|
| `sacrifice`, `sacrifices`, `dies` | `strategy.sacrifice` |
| `token`, `tokens` | `strategy.tokens` |
| `cast`, `instant`, `sorcery`, `magecraft`, `storm` | `strategy.spellslinger` |
| `graveyard`, `return` + `graveyard`, `reanimate` | `strategy.reanimator` |
| `commander damage`, `equipped`, `aura` | `strategy.voltron` |
| `+1/+1 counter`, `proliferate` | `strategy.counters` |
| `landfall`, `land enters` | `strategy.landfall` |
| `tribal`, creature type name repeated in type line context | `strategy.tribal` |
| `artifact` | `strategy.artifacts` |
| `enchantment` | `strategy.enchantress` |
| `counter target`, `draw`, `cannot be countered` | `strategy.control` |
| `flicker`, `exile` + `return`, `enters the battlefield` | `strategy.blink` |
| `planeswalker`, `loyalty` | `strategy.superfriends` |
| `gain control`, `steal` | `strategy.theft` |

**Win-condition signals (weaker in v1 — prefer fallback):**

| Keyword / phrase | Boosts |
|------------------|--------|
| `mill`, `library` + `graveyard` | `wincon.mill` |
| `lose life`, `drain`, `lifelink` | `wincon.life_drain` |
| `infinite`, `win the game`, `you win` | `wincon.combo` |
| `can't`, `prevent`, `skip` + `phase` | `wincon.lock` |
| `commander damage`, `combat damage` | `wincon.commander_damage` / `wincon.combat` |

Return top `PLAN_PRIMARY_OPTIONS_COUNT` by score; if top score &lt; `PLAN_INFERENCE_CONFIDENCE_MIN`,
use static fallback list.

##### Deck analysis ranking (`rankForDeck`)

For decks with `cardCount >= PLAN_WIZARD_ANALYZE_THRESHOLD`:

1. Build deck signal vector from **role-tag counts** (project tag IDs) and **card-type
   ratios** (creatures, instants/sorceries, artifacts, enchantments, lands).
2. For each strategy ID, sum `STRATEGY_DECK_SIGNALS[strategyId]` × signal value ×
   `PLAN_TAG_SIGNAL_WEIGHT`.
3. For each win-condition ID, sum `WINCON_DECK_SIGNALS[winconId]` × signal value ×
   `PLAN_TAG_SIGNAL_WEIGHT` (smaller signal table — v1 uses fewer win-con deck signals).
4. Normalize scores to 0–1 by dividing by max possible for that catalog.
5. Return top `PLAN_PRIMARY_OPTIONS_COUNT`; if top &lt; `PLAN_INFERENCE_CONFIDENCE_MIN` →
   static fallback.

**`STRATEGY_DECK_SIGNALS` (v1 — map to project role-tag IDs at implementation):**

| Strategy ID | Signals (semantic — map to project tags in repo) |
|-------------|--------------------------------------------------|
| `strategy.tokens` | token-producing tags, creature tokens, go-wide |
| `strategy.sacrifice` | sacrifice outlets, dies triggers, aristocrats |
| `strategy.spellslinger` | instant/sorcery density, cast triggers, prowess |
| `strategy.reanimator` | recursion, graveyard recursion, reanimate |
| `strategy.voltron` | equipment, auras, commander-centric |
| `strategy.counters` | +1/+1 counters, proliferate |
| `strategy.landfall` | landfall, land recursion |
| `strategy.tribal` | dominant creature type share &gt; 40% |
| `strategy.artifacts` | artifact density, artifact synergy tags |
| `strategy.enchantress` | enchantment density, enchantress tags |
| `strategy.control` | counterspell, removal, draw density |
| `strategy.blink` | ETB, flicker |
| `strategy.superfriends` | planeswalker count |
| `strategy.theft` | steal, gain control |

**`WINCON_DECK_SIGNALS` (v1 — sparse):**

| Wincon ID | Signals |
|-----------|---------|
| `wincon.combat` | high creature count, combat keywords (trample, haste) |
| `wincon.commander_damage` | voltron signals + commander-centric |
| `wincon.combo` | tutors + instants, known combo enabler tags |
| `wincon.mill` | mill cards, library manipulation to opponent GY |
| `wincon.life_drain` | drain, lifelink, ping |
| `wincon.lock` | stax, prison, resource denial tags |
| `wincon.value` | high draw + removal (control grind) |

Chip inference = rank #1 if score ≥ `PLAN_INFERENCE_CONFIDENCE_MIN`; else chip omitted or
shown as "uncertain" (user must pick).

##### Plan-aware backfill (entry 5 integration)

**Gate change:** allow unowned fetch when largest active deficit is **Plan** AND deck has
`winConditionId` + `primaryStrategyId` set (wizard complete or user set plan fields).

**Filter:** candidate must be Plan-tagged (no utility role tag) OR explicitly in plan
candidate pool; must match declared plan via `planMatchScore`:

```
planMatchScore(card) =
  strategyMatch(card, primaryStrategyId) * 2
  + strategyMatch(card, secondaryStrategyId) * 1  // if set
  + winconMatch(card, winConditionId) * 1
```

`strategyMatch` / `winconMatch`: 0 or 1+ from `PLAN_BACKFILL_SIGNALS` — oracle keyword /
type / tag overlap tables keyed by strategy/wincon ID (same vocabulary as commander rules;
implementation builds lookup from catalog IDs).

**Rank:** sort filtered candidates by `planMatchScore` desc, then existing Adds score
(equal-weight Plan — no change to deficit formula; planMatchScore only affects ordering
within Plan backfill pool).

**Archetype:** when plan fields set, do not use archetype for plan backfill path (#28).

##### Schema (v1)

```json
{
  "winConditionId": "wincon.life_drain",
  "primaryStrategyId": "strategy.sacrifice",
  "secondaryStrategyId": null,
  "fieldSources": {
    "winConditionId": "chip-confirmed",
    "primaryStrategyId": "formal"
  }
}
```

v2 hooks: `tertiaryStrategyId`, `hybridRoleModifiers`, `cutsShielding` — nullable, unused in v1.

##### Verification cases (v1)

| # | Scenario | Expected |
|---|----------|----------|
| 1 | ≥80-card sacrifice/aristocrats deck, wizard run | Chips infer `strategy.sacrifice` + `wincon.life_drain` or `wincon.combat` with score ≥ threshold; confirmed chips skip formal Q |
| 2 | &lt;80 cards, Korvold commander | `strategy.sacrifice` in top 6 from oracle rules |
| 3 | Plan-only deficit, plan declared | Unowned fetch triggers; results match sacrifice/drain signals not random untagged |
| 4 | Plan-only deficit, plan **not** declared | No unowned fetch (current behavior) |
| 5 | Inference score &lt; 0.35 | Static fallback lists shown; no false-confident chip |
| 6 | User skips chip, formal Q | Pre-filled with inference when score ≥ threshold |
| 7 | User navigates back | Prior answers editable; schema updates |

---

## Agent prompt: Entry 13 v1 — plan wizard + plan-aware backfill

**Runnable copy:** Prompt **2 of 3** in
[`cuts-adds-ready-prompts.md`](./cuts-adds-ready-prompts.md)
(twin: [`entry-13-v1-implementation-prompt.md`](./entry-13-v1-implementation-prompt.md)).

Do **not** move Entry 13 to `cuts-adds-archive.md` until Status reaches **Shipped**.

---

**Design philosophy summary:** ensure the deck is fundamentally healthy first, then help
make it uniquely *this* deck. Functional roles are essential; Plan gives personality. The
wizard captures that personality in structured, deterministic form for the existing
recommendation algorithm — without AI at runtime.

### Coordinated scoring pass — entries 7 + 9 + 10 + 11 + 12
- Status: **Prompt drafted** (2026-07-12) — **Runnable copy:** Prompt **1 of 3** in
  [`cuts-adds-ready-prompts.md`](./cuts-adds-ready-prompts.md). Ship as **one agent task**,
  not five separate PRs. Entries 7, 9, 10, 11, 12 remain individually tracked above; this
  entry is the integration record.
- Side: Adds (+ server precompute for E)
- Confirmed cause: spec-level only; repo agent must verify D/V/C formulas and tag IDs
  before editing.
- Proposed fix: see **Agent prompt** block below. Final formula:
  `Score = (D × M) + C_eff + L + E + B − P + V + T + K`
  Weight order: `D, M` > `C or L` > `E` > `B` > `P` > `V` > `T, K`.
- Constraints: Adds-only scoring. Do not touch Cuts, candidate pool, `tribes: []`, CK gate.
  Entry 11 tag mapping must use **project role-tag IDs**, not Scryfall `otag:` slugs
  directly. Entry 1 (commander CMC in Adds curve) is a separate task unless user
  explicitly bundles it.
- Open questions: Spellslinger detection signal — repo must document. (Tier 2 tags for
  entry 11 — decided yes, see entry 11.)

---

## Agent prompt: Coordinated Adds scoring rebalance (entries 7/9/10/11/12)

Copy everything in this fenced block to an agent with repo access:

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
Plan/untagged, land-ramp categories — see entry 11 exclusion table.

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

## Intake: new symptoms go here first
Use this section to drop in raw observations ("deck X got suggested to cut Y, which was
obviously wrong because Z") before they're triaged into a numbered entry above.

-


---

## Project context: Suggested Cuts / Suggested Adds feature

**Purpose of this project:** making this feature better, in two ways — fixing suggestions
that are bad or flat-out incorrect, and building enhancements/new capabilities it doesn't
have yet (e.g. new scoring signals, new UI controls). Treat descriptions below as the
current (possibly buggy, possibly incomplete) behavior, not the desired end state — don't
assume something is correct or complete just because it's documented here.

**Implementation model:** all recommendation behavior — current scoring, planned scoring
changes (entries 7–12), and the Entry 13 plan wizard — is a **deterministic algorithm**
(formulas, rules, structured input). The shipped product does not use AI at runtime.
Coding agents are used to author that code; do not propose LLM/ML dependencies as part
of the feature unless the user explicitly reopens that constraint.

This project stays scoped to Suggested Cuts / Suggested Adds only, not the wider
deck-builder app. This doc is for the user only, not a shared team reference.

## Domain model
- Cards are tagged with ~36 role tags (Ramp, Card Draw, Removal, Board Wipe, Tutor,
  Counterspell, Protection, Recursion, etc.), sourced from Scryfall's community tagging
  project, cached server-side, user-overridable per card and per deck.
- Baseline "ideal recipe" for a 100-card Commander deck: Ramp 10, Card Draw 10,
  Removal 10, Board Wipe 3, Tutor 2, Counterspell 3, Protection 3, Recursion 3,
  Plan 30 ("Plan" = no utility tag).
- Recipe is adjusted by: (1) detected/overridden **archetype**, (2) the **Aggro↔Control
  slider** (one slider, drives both Cuts and Adds thresholds), (3) an **ideal mana curve**
  (format + land ratio + ramp + commander CMC → Gaussian-blended target curve).

## Cuts (decks.js:6254 `_suggestCardsToCut`)
- Only shown when deck qty > 100. Candidates = deck cards minus commander/tokens/lands.
- Score = max role surplus (over threshold) + CMC factor + "competes with commander"
  CMC penalty + roleless-card bonus − multi-role discount + cheap-price bonus + curve
  bloat penalty − tribal shield − commander-theme shield + dead-payoff gate penalty.
- Top 5 shown. **No hard conditional-keyword gate** — dead payoffs only soft-penalized.

## Adds (decks.js:6623 `_renderAddSuggestions`, async)
- Shown whenever deck has cards. Candidate pool: owned collection first (need color
  identity fit, not already in deck, a free/unallocated copy); if <8 owned qualify and
  a real (non-Plan) deficit exists, backfill from server `/api/cards/by-roles` (unowned,
  local DB only, never live Scryfall, `tribes` sent empty on purpose).
- Score = role deficits filled × dead-payoff multiplier (0.3–1.0) + curve-gap bonus +
  versatility bonus + tribal bonus + commander-theme bonus. Roleless filler capped at +3.
- **Hard gate**: conditional-keyword mechanics (Prowess, Delirium, Threshold, etc.)
  need ≥15 qty-weighted enablers in the deck or the candidate is silently dropped —
  no matter its score, no "N hidden" note (unlike the EDHREC recs panel).
- Top 8 shown (`_ADD_SUGGESTION_COUNT`), owned-first.

## Known quirks/inconsistencies

Status key: **CONFIRMED BUG (fix)** = user has directed a fix. **FLAG ONLY** = don't touch
without explicit instruction, just make sure it's on the radar. **INTENTIONAL, DO NOT TOUCH**
= deliberate design choice by user's partner, leave alone unless discussed.

1. **FLAG ONLY, likely unintentional but unconfirmed.** Plan-count token exclusion differs:
   Cuts' candidate pool already excludes tokens, so tokens fall out of its Plan-count
   math as a side effect. Adds' Plan-count filter doesn't also exclude tokens. Never
   explicitly documented as intentional — verify in actual code before treating as
   settled either way.
2. **CONFIRMED BUG — Prompt drafted.** Curve bucket commander-inclusion differs:
   Cuts includes the commander in curve calc; Adds excludes it. **Adds should be changed
   to also include the commander's CMC in the curve calculation, matching Cuts.**
   Runnable prompt: Prompt **3 of 3** in `cuts-adds-ready-prompts.md`. Do not move to
   archive until Status reaches **Shipped**.
3. **FLAG ONLY — important for future work.** `_deckSwapsEnabled(deck)` is called with a
   `deck` arg in both renderers but the function takes no parameters (toggle is user-wide,
   not per-deck). Harmless today, but any code touching Adds/Cuts planning-mode behavior
   needs to know this.
4. **INTENTIONAL, DO NOT TOUCH — user's partner's decision.** Adds always sends
   `tribes: []` to the server by design (tribal matching used to be weighted too heavily
   and over-tuned suggestions). Don't "fix" without discussing with user first.
5. **FLAG ONLY, informational.** A deck deficient only in Plan cards never triggers the
   unowned-card fetch — likely explains "no suggestions" on decks well-stocked on roles
   but short on theme cards. See entry 5 (blocked on entry 13).

## Key file/line anchors (may drift — verify against current code before citing)
- decks.js:6254 `_suggestCardsToCut`, decks.js:6489 `_scoreAddCandidate`,
  decks.js:6190 `_computeBaseThresholds`, decks.js:6216 `_computeCutThresholds`,
  decks.js:7126 `_computeIdealManaCurveContext`, decks.js:11149 `CK_REQUIRED_ENABLERS` (15).

## How to use this
- **Algorithm, not AI:** implement and extend this feature as deterministic code only.
  See "Note for coding agents" at the top of this doc. Do not add runtime LLM/ML calls.
- Treat this as ground truth for how scoring currently works — don't re-derive from
  first principles or guess at intent for the quirks above.
- If asked to change scoring behavior, name which side (Cuts vs Adds) and which term
  in the formula is being changed, since the two panels share some state but diverge
  in several scoring terms (see table above).
- This project covers both bug fixes and enhancements/new features for Cuts/Adds. A
  backlog entry doesn't need an observed bad suggestion to be valid — a feature request
  with a clear proposed behavior is tracked the same way, through the same Status
  pipeline, and held to the same bar before a prompt is drafted (Fix scoped or later).

---

**Working backlog doc (Cuts/Adds Improvement Backlog)**

There is a separate working document (uploaded to Project knowledge) that tracks
debugging findings and enhancement/feature requests, turning both into scoped agent
prompts. It is distinct from these instructions: this doc is fixed ground truth, the
backlog doc is a living list of in-progress issues and in-progress improvements. A third
doc, `cuts-adds-archive.md`, holds closed/ruled-out/shipped entries with full reasoning.

*Adding to the doc:*
- When the user says "add X to the doc" or drops something in the doc's Intake section,
  read the current uploaded version of the backlog doc first, then edit it.
- New raw symptoms should be **triaged**, not just appended: convert them into a
  numbered backlog entry using the doc's existing Entry template, filled in as far as
  can be determined, with genuinely unknown fields marked TBD rather than guessed.
- Always return the **full updated document**, not a diff or a snippet.
- Don't advance an entry's Status past what's actually been confirmed.
- Respect this doc's own constraints fields and the known-quirks list — don't let a
  triage pass quietly reverse something flagged as intentional or flag-only.
- When an entry reaches Shipped or is ruled out/closed, move its full write-up to
  `cuts-adds-archive.md` and leave a one-line pointer in the active doc.

*Drafting a prompt from an entry (only on explicit request):*
- Only draft a ready-to-use prompt for entries at **Fix scoped** or later. If an entry
  isn't there yet, say so and offer to help investigate/scope it instead.
- **State explicitly in every prompt:** the deliverable is deterministic algorithm code;
  no runtime AI/LLM dependencies. Coding agents write the code; the product does not use AI.
- Follow the doc's "How an entry becomes a prompt" template: verified current behavior +
  file/function, the defect scoped to one term on one side, the desired change, an
  explicit do-not-touch list, and a verification step.
- Write it for an autonomous coding agent with no memory of this conversation — fully
  self-contained, no "as we discussed" references.
- Present it in a clearly delimited block so it's easy to copy out.
- Keep it tight — scoped to the one entry, not a recap of the whole project.
