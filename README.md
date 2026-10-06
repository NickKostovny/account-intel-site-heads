# account-intel-site-heads

Stage 3 of 5 in Invert's account-intelligence pipeline. Turns the prose "named site head" from the site-mapping work into a Clay-verified person with a LinkedIn link, current title and work email. Upstream: [account-intel-core](https://github.com/NickKostovny/account-intel-core). Downstream: the briefs. Canonical code: [account-intelligence](https://github.com/NickKostovny/account-intelligence).

## Why this stage exists

The site-map sheets carried a `named_site_head` column written as prose, with placeholders (`PENDING-CLAY` on 156 sites, `n/a`, "Not identified", "None surfaced", site-closed notes). A raw non-empty count said 531 named heads; the real number was 301 of 604. For the 2026-09-10 sales-facing pass the head of sales needed a person a rep can click, not a sentence. So the names became a table of their own, verified through Clay.

## Files

| File | Role |
|---|---|
| `clay_targets.py` | pulls proper names out of the `named_site_head` prose on KEEP sites of active accounts. A stoplist rejects org words; the title must contain a title keyword. Ranks by pipeline status (P1 live deal, P2 no deal, P3 closed lost). Writes `data/clay_targets.csv` and 20-name batch files under `data/cache/clay/batches/`. |
| `merge_clay.py` | folds the Clay batch results (`data/cache/clay/batchNN.json`) into `data/site_heads.csv`. Upsert by `site_head_id` (the target id). Not-found rows are kept with `found=false`. Unescapes HTML entities. Applies the email hygiene below. |
| `merge_roles.py` | folds role-search results (`data/cache/clay/roles/*.json`) for sites whose head was `PENDING-CLAY`. |
| `aiq.py` | shared library, copy of the canonical one; this stage uses `aiq.DOMAIN` (curated company domains) and `aiq.EMAIL_DOMAINS` (live subsidiary and legacy domains). See `SHARED.md`. |

## The process

1. `python3 clay_targets.py`. 189 people on 39 accounts in the first run.
2. For each batch file, a subagent calls the Clay MCP `search-contacts-by-name` once (workspace 1243060, 20 names per call) and saves the result to `data/cache/clay/batchNN.json`.
3. For each found person, `add-contact-data-points` with the Email data point.
4. `python3 merge_clay.py`, then `python3 build-briefs.py` in the briefs stage.

The brief renders a found person as a LinkedIn link with Clay's current title and "matched on LinkedIn <date>". The site-map prose stays as the fallback, labelled "site-map evidence". Only `found` rows render.

**Email hygiene.** An initials-only local part (`m.c@` or `ir@` style) is `email_status=suspect`. An address whose domain is neither the curated company domain nor a listed subsidiary or legacy domain is `suspect_domain` (an old parent's domain at an acquired company, for example). Both stay in the table; neither renders.

**Role search for PENDING-CLAY sites.** The Clay DSL accepts only `headline`. One agent per account, a ranked site list and location strings, the Email step, a fixed JSON schema, written to `data/cache/clay/roles/<account>.json`, then `python3 merge_roles.py && python3 build-briefs.py`.

## Verified numbers (2026-09-10)

| | checked | on LinkedIn | emails returned | usable |
|---|---|---|---|---|
| All | 189 | 147 (78 percent) | 127 | 120 |
| P1 live deals | 66 | 50 | | 44 |
| P2 no deal | 74 | 63 | | 46 |
| P3 closed lost | 49 | 34 | | 30 |

Held back: 3 initials-only addresses and 4 acquired-domain addresses. Zero Clay credit warnings.

## Gotchas

- Guessing a domain finds nothing (`bristolmyerssquibb.com` returned no one). Use `aiq.DOMAIN`.
- The Clay connector can be pinned to an empty workspace; credits live in 1243060. Enrichment is async; re-poll `get-task-context`.
- Per-user credit cap is 1,000; 20 contacts per call.
- Pulling people data into any LLM other than Claude (for example TypeSafe) needs Nick's explicit approval. Not given as of this handoff.

## Open

- PENDING-CLAY heads for Eli Lilly (11 sites) and Novo Nordisk (15 sites). The agents were stopped before writing any role files on 2026-09-10. Candidate names for Lilly are listed in the umbrella repo's HANDOFF.md §0.
- Site-scoped `search-contacts` for the remaining PENDING-CLAY sites (116 on active accounts, all EU at last count).
- Clay contact waterfall on the full people table.

## Data

Not in this repo. `data/clay_targets.csv`, `data/site_heads.csv` and `data/cache/clay/` name real people. Column lists are in `TABLES.md`.
