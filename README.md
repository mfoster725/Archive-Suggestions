# Archive-Suggestions

Docs-only archive for the Cuts/Adds improvement work.

| File | Purpose |
|------|---------|
| `cuts-adds-backlog.md` | Living backlog, design interviews, algorithm specs |
| `cuts-adds-archive.md` | Closed / ruled-out / **shipped** entries |
| **`cuts-adds-ready-prompts.md`** | **Ready-to-run agent prompts, in implement order** |
| `entry-13-v1-implementation-prompt.md` | Twin of Prompt 2 in the ready-prompts doc (Entry 13 v1) |

## Partner handoff

1. Open **`cuts-adds-ready-prompts.md`**.
2. Run prompts **in listed order** (currently: 1 → scoring rebalance 7/9/10/11/12, then 2 → Entry 13 v1).
3. Each prompt runs in the **main deck-builder repo** (`decks.js`), not here.
4. After ship: move entry to archive; remove that prompt from the ready-prompts queue.
