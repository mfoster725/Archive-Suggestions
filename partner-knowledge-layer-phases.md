# Partner knowledge layer → recommendation engine (Phases 3–5)

**Status: In progress — partner workstream. Not done.**

This doc records what the **partner** is building in the main deck-builder app. It is
**not** an Archive-Suggestions implementation prompt and **not** ready-to-run work for
coding agents in this docs repo.

## How this relates to Cuts/Adds work here

What's tracked in this archive (`cuts-adds-backlog.md`, ready prompts, Entry 13 wizard,
scoring rebalance, etc.) **complements and helps** the Suggested Adds/Cuts panels. The
partner's Phases 3–5 plan below is the longer-horizon path that can eventually give those
panels a richer, axis-aware brain. Neither stream replaces the other:

| Stream | Owner | What it does |
|--------|--------|--------------|
| **This archive** | Our Cuts/Adds backlog & agent prompts | Scoped, shippable improvements to current Suggested Adds/Cuts (scoring terms, plan wizard, etc.) |
| **Partner phases below** | Partner (main app) | Knowledge extraction + deterministic interaction/goal/recommender stack that can later swap in behind the same Adds/Cuts UI |

Treat partner phases as **context and future integration**, not as blockers for the prompts
in `cuts-adds-ready-prompts.md`.

---

## What is running now (knowledge layer only)

What's running now **builds the knowledge layer**. It does **not** recommend anything by
itself.

After the pilot / corpus extraction runs, every card is structured data:

- Abilities as typed effects
- Capability axes in `card_semantics_axes`

Example: Viscera Seer **provides** `creatures_dying`; Blood Artist **needs**
`creatures_dying`. That data sits inert in MySQL until downstream code joins and scores it.

The expensive, previously-impossible part — a machine actually understanding ~27k cards —
is what the extraction runs buy. Recommendation logic is separate.

---

## Recommendation logic (separate, deterministic)

The recommendation path is **deterministic code** (no LLM, no credits). It reads the
knowledge layer. It is **Phases 3–5 of the partner plan**: fully specified, **not yet
written**.

None of these phases touch an LLM again. They are ordinary pure-JS coding sessions with
fixture tests.

### Phase inventory

| Phase | Module(s) | Role | Status |
|------:|-----------|------|--------|
| (done / running) | Extraction → MySQL | Card axes & typed effects (knowledge layer) | In progress / pilot usable |
| **3** | `engine2/interactions.js` | Synergy edges from axes | Specified, not written |
| **4** | `deck-goals.js` | Goal inference from axes + edges | Specified, not written |
| **5** | `recommender.js` + `POST /api/decks/analyze` + `explain.js` | Cuts/Adds scoring + explanations; wire into existing panels | Specified, not written |

---

### 1. What you have after pilot / corpus runs

Every card as structured data — abilities as typed effects, plus the capability axes in
`card_semantics_axes`. That is **inert data** in MySQL until Phase 3+.

---

### 2. Phase 3 — interaction engine (`engine2/interactions.js`)

Joins those axes into synergy edges:

- Enabler → payoff pairs
- Engines (cycles like token maker → sac outlet → death payoff)
- Curated combo signatures (e.g. Thoracle + Consultation)
- Nonbos (e.g. your own Rest in Peace vs your reanimation package)

Implementation sketch: pure hash-joins over the axes; target **sub-100ms per deck**.

**Nice property for incremental work:** Phase 3 does **not** need the full corpus. The
pilot's **708 cards** cover all **12 test decks** completely, so the interaction engine can
be built and tested against **real extracted data** now (e.g. Korvold's deck should light
up with death-trigger edges) while Opus requeue and the full corpus run continue in the
background.

---

### 3. Phase 4 — goal inference (`deck-goals.js`)

Reads:

- Commander's axes (weighted **3×**)
- The deck's axis histogram
- Synergy clusters from Phase 3

Ranks goal hypotheses (e.g. `"aristocrats, confidence 0.86"`) and sets role thresholds per
goal.

---

### 4. Phase 5 — the recommender (`recommender.js` + `POST /api/decks/analyze`)

1. Score every deck card's contribution (synergy edges + role fill + curve fit + goal
   alignment) → lowest contributors become **cuts**.
2. Query the axes table for commander-legal cards that provide the deck's **deficit axes**
   → score with collection preference and prices → **adds**.
3. `explain.js` turns reasoning traces into human-readable strings (e.g. *"Feeds Blood
   Artist + 3 other death payoffs"*).
4. Existing Suggested Adds/Cuts panels get their brains swapped to call this endpoint,
   still feeding the **same planning board** already in use.

---

## Bottom line

- Knowledge extraction = costly/hard part, underway.
- Phases 3–5 = ordinary deterministic JS + fixtures; **no further LLM at recommendation
  time**.
- This archive's Adds/Cuts work **helps and complements** that path; partner work is **not
  finished** and should not be treated as shipped Suggested Adds/Cuts behavior yet.

## Open partner question (logged, not actioned here)

Partner asked whether to start on `interactions.js` next (Phase 3), given the pilot already
covers the 12 test decks. That decision lives with the partner / main-app owners — not an
Archive-Suggestions prompt.
