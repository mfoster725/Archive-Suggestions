# Cuts/Adds Archive (closed / ruled-out / shipped entries)

This doc holds full reasoning for entries that are closed, ruled out, or shipped.
The active backlog (`cuts-adds-backlog.md`) references these by ID with a one-line
summary — pull this doc into context only if an entry needs to be re-litigated.

---

### 8. Per-role card quality signal (primary vs. secondary tag) — investigated, ruled out
- Status: Investigated — closed, no viable fix identified. **Closed harder as of
  2026-07-11: EDHREC-sourced solutions (including a narrowed "multi-tag tiebreaker
  only" variant) are off the table on legal grounds, not just data-quality/maintenance
  grounds — see Confirmed cause #1 update below.**
- Side: Both
- Symptom: n/a — investigation prompted by entry 7. A card tagged with multiple roles
  (e.g. both Ramp and Card Draw) has a single blended EDHREC popularity number, with no
  way to tell which tag is actually driving that popularity. Example: a card that's
  mediocre at ramp but well-loved for its card draw would look identically "good" in
  both buckets under a naive rank-based scoring approach.
- Suspected cause: n/a
- Confirmed cause: Investigated two candidate data sources, both ruled out:
  1. **EDHREC's per-category pages** (e.g. edhrec.com/top/ramp) — no official API;
     using this would require unofficial/unversioned `json.edhrec.com` endpoints, a
     bigger departure from the existing "local DB, cached from Scryfall" pattern and
     from the "never live Scryfall" design principle already in place for Adds.
     EDHREC's own FAQ also states category definitions aren't rigorously or
     consistently defined across categories.
     **Update 2026-07-11 — ToS violation, not just a design/quality concern:**
     reviewed EDHREC's Terms of Service (edhrec.com/terms, effective 2024-08-06)
     directly. Section 2 ("Access to the Site") grants a personal/noncommercial-use
     license only and explicitly prohibits (a) commercially exploiting the site, (b)
     accessing the site to build a similar/competitive site, and (c) copying,
     reproducing, downloading, or distributing any part of the site except as
     expressly permitted. Section 3's Acceptable Use Policy separately prohibits
     "automated agents or scripts... to generate automated searches, requests, or
     queries to the Site." A periodic scraper/poller building a local DB from
     `json.edhrec.com` or the rendered category pages is exactly this kind of
     prohibited automated querying, and storing the results locally runs into the
     no-copy/no-distribute restriction too. **This is an access-method and
     reproduction problem, not a data-use problem** — narrowing the use case (e.g. to
     a multi-tag tiebreaker only, using only relative order rather than absolute
     rank) does not avoid it, since the violation is in the querying/storage
     mechanism itself, not in how the resulting number is applied downstream. Any
     future EDHREC-sourced local DB for this project is ruled out on this basis, not
     just deprioritized as a maintenance liability.
  2. **Scryfall Oracle Tags bulk file `weight` field** (per card-tag "tagging") —
     confirmed via direct inspection of real bulk data (2026-07-10) that `weight` is
     not a continuous or fine-grained score. It's a small discrete value set (observed:
     `median`, `very_strong`; likely also lower/higher tiers not observed in the sample
     pulled) that is overwhelmingly defaulted to `median` — e.g. 557/557 "ramp"
     taggings were `median`, and 1707/1708 "removal-destroy" taggings were `median`
     with exactly one `very_strong` outlier (Murder). This indicates `weight` functions
     as a rarely-touched manual override rather than a routinely-assigned prominence
     judgment, and cannot reliably distinguish a card's dominant role from an incidental
     one.
- Proposed fix: none — no viable data source currently identified for isolating
  per-role card quality/prominence. EDHREC is excluded as a source outright (ToS);
  Scryfall `weight` is excluded on data-quality grounds. Not being pursued further
  unless a new, ToS-compliant data source is suggested. **Mitigation path (not a
  reopen of #8):** entry 7's contextual E (per-tag percentile, dampened for multi-tag
  cards) + entry 9's castability signal (P) + entry 10's versatility rebalance + entry
  11's CMC efficiency (L) are the approved substitute approach — see those entries
  rather than reopening this one.
- Constraints: Any future proposal must not rely on scraping or polling EDHREC (site
  or `json.edhrec.com`) in any form, including one-off/manual pulls at scale, per ToS
  Sections 2–3.
- Open questions: none currently — closed pending any new, ToS-compliant data source
  suggestion.

**Note (from project memory):** Johnny liked the original proposal to use Scryfall's
Oracle Tags `weight` field to resolve primary role tags, pending real bulk data
confirmation — which subsequently ruled it out. The appeal of the approach is noted
even though it was invalidated by evidence.

---

## Shipped entries

*(none yet — move entries here from the active backlog once their Status reaches
Shipped, keeping full reasoning for reference. Active doc should retain only a
one-line pointer.)*
