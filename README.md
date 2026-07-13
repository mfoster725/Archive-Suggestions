# Archive-Suggestions

Docs-only archive for the Cuts/Adds improvement work.

| File | Purpose |
|------|---------|
| `cuts-adds-backlog.md` | Living backlog, design interviews, algorithm specs |
| `cuts-adds-archive.md` | Closed / ruled-out / **shipped** entries |
| **`cuts-adds-ready-prompts.md`** | **Ready-to-run agent prompts, in implement order** |
| `entry-13-v1-implementation-prompt.md` | Twin of Prompt 2 in the ready-prompts doc (Entry 13 v1) |
| **`partner-knowledge-layer-phases.md`** | **Partner WIP (not done):** knowledge layer + Phases 3–5 recommender plan; complements Suggested Adds/Cuts |

## Partner handoff (ready prompts)

1. Open **`cuts-adds-ready-prompts.md`**.
2. Run prompts **in listed order** (currently: 1 → scoring rebalance 7/9/10/11/12, then 2 → Entry 13 v1).
3. Each prompt runs in the **main deck-builder repo** (`decks.js`), not here.
4. After ship: move entry to archive; remove that prompt from the ready-prompts queue.

## Partner knowledge layer (separate, in progress)

See **`partner-knowledge-layer-phases.md`**. Extraction builds inert card axes; Phases 3–5
(interaction engine → goals → recommender) are specified but not written. Work in this
archive **complements** that plan and continues to improve Suggested Adds/Cuts on its own
track — do not treat the partner phases as shipped or as blockers for the ready prompts.
