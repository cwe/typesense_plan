# State of play (13 October 2026)

Where everything is, and what has been measured. Facts only; the questions are in the chunks.

## Code

- Repo `cwe/resourcespace`, branch **`typesense`**, tip `c72fe64e` (30 Sep, "keyword fragments
  experiment"). The plugin is `plugins/typesense_search/`.
- The branch carries one **core patch**: `include/do_search.php` — the `external_search` hook's
  return handling is enabled (it was commented out in the original). Without it the plugin's results
  are ignored. See chunk 11.
- **Staged, uncommitted, in the main checkout** (currently on branch
  `claude/typesense-search-review-c73740`): the inactive-field fix — a field-scoped search on an
  inactive field vetoes, the attribute query takes `rtf.active = 1`, lookup returns `active`
  (3 files: `typesense_search_functions.php`, `typesense_search_query.php`, the architecture doc).
  Needs a plain reindex. Another session's work; leave it.
- Two **local branches, not pushed** (Chris sorts out pushing):
  - `typesense_search_harness` (off `typesense` @ `c72fe64e`): `plugins/typesense_search/harness/`
    (local A/B batteries, trace harness, live API runner, combinations) and
    `docs/typesense-search-review.md`.
  - `typesense_search_catalogue` (off the harness branch): `docs/core-search-catalogue.md`,
    `docs/search-gaps.md` (committed), `docs/search-access-gaps.md` (staged only), the catalogue
    runner and table generator.
- Deployed on the test RS host: `typesense` @ `c72fe64e`, index rebuilt with fragments on 30 Sep.
  The inactive-field fix is **not** deployed.

### Plugin architecture (built)

Pipeline: Context → Parse → one **Mode** (standard, collection, last, list/listall, resource_ref,
contributions, hasdata, archive_pending, user_pending, unsupported_special) → shared keyword/node
step → **Restrictions** (standard, featured_collections `J`, group_filter, access) → default sort →
Compile → Execute (multi_search windows) → Hydrate (MySQL rows by ref, Typesense order).
Config toggles: enabled, Typesense-only, indicator badge, max rows (25,000), filter max ops (100),
import chunking.

### Index (Typesense 30.2, three collections per prefix)

- `resources`: ref, ref_s, title, resource_type, archive, created_by (optional), access,
  creation_date, date_field_sort (text sort key), modified_date, nodes[], populated_field_ids[],
  and per field `field_<ref>_{s,ss,text,q,f,ts,range_start,range_end,p}`. Text fields stem iff
  `$stemming`; `token_separators` from `$config_separators`; invisible characters stripped;
  partial-index fields get `_p` fragments.
- `resource_collection_memberships`: `collection:resource` ids, collection_ref, sortorder (NULL →
  INT32_MIN), date_added, collection_type. Selection, upload and share-upload collections are not
  indexed.
- `resource_access_grants`: user / usergroup (−1 when unused), access, expires (checked at query time).

## Documents (in `plugins/typesense_search/docs/`)

| File | Branch | What it is |
|---|---|---|
| `resourcespace-search-behaviour.md` | typesense | Core search behaviour as a spec (§13: combination matrix with plugin coverage) |
| `typesense-query-architecture.md` | typesense | The pipeline, status, config dependencies, verification items 1–19 |
| `typesense-incremental-sync-plan.md` | typesense | Sync design: 13 decisions taken, phases 0–3, event inventory, Typesense facts |
| `typesense-collection-parity-plan.md` | typesense | `!collection` / `J` parity: phases, open decisions, core write sites |
| `typesense-search-review.md` | harness | 48 divergences in themes A–H, Core 1–4, Appendix C (Core 5–10, B18–B22, E11–E14, H6–H8) |
| `core-search-catalogue.md` | catalogue | Every core search form as a table (groups A–K, ~300 rows, measured live) |
| `search-gaps.md` | catalogue | Behaviours inconsistent with themselves (core and plugin); three decisions |
| `search-access-gaps.md` | catalogue (staged) | Where core uses a search as an access gate, and the plugin's coverage |

A meeting copy of the review exists as a Claude doc (keep in step with the repo copy).

## Tests

- **Local A/B harness** (`harness/ab/`): SQLite stands in for MySQL, a local Typesense 30.2 runs on
  a private port, the real core `do_search()` and the real plugin run twice per case. 30 batteries
  (~12 s) plus `50_combinations.php` (factorial fixture, 2,481 generated cases, ~2.5 min). SQL
  emulation limits: full-text `MATCH … AGAINST` is emulated; accents are not.
- **Live API A/B** (`harness/live/`): the RS API as two users in two groups (one group with the
  plugin in Typesense-only mode, one with it disabled); credentials from environment variables
  only. ~45 paced searches in ~3 minutes; core date-field searches take 6–8 s each.
- **Catalogue runner**: 269 API cases, ~1 hour with 5 s pauses.
- **Trace harness** (`harness/trace/`): the earlier stubbed-core harness.
- Scratchpad harnesses outside the repo are deleted by the OS after ~3 days; several were lost.

## Test systems

- The **test RS host** runs RS and Typesense 30.2 on one 16 GB machine. It overheated and shut down
  under back-to-back heavy MySQL searches (29 Sep): no AND chains above ~40 nodes, no big `IN` lists,
  space heavy searches out, ask before minutes-long runs.
- Its database (YSP): 114,596 resources (access 0/1/2 = 113,136 / 1,387 / 73), 155,141 memberships
  under the indexing rule, 6 grants; `$stemming` on; date field 12; one partial-indexed field (51,
  original filename); no category trees, HTML fields, translated values or search filters; the only
  related-keyword pair is antony/anthony; both API test users hide states 1–3 (`z1 z2 z3`); numeric
  test field 170 holds 12 values.
- A seeded test system exists (the test-system seeder tool) for data the YSP database lacks.
- Local machine: Typesense 30.2 binary, PHP 8.5, no MySQL.

## Measured headline figures

- Full rebuild: just over 1 minute for ~115k resources + ~152k memberships; resident ~800 MB with
  fragments (+78 MB for fragments).
- Result windows: 25k rows 1.5–4 s, whole index 20–37 s (deep pages re-rank).
- Node filters: 600/1,000-id OR buckets served; a non-`v` user's AND chain served to 40 nodes,
  vetoed from 41 (restrictions cost 19 of the 100 ops); core itself fails at 58 joins.
- Parity (live, 29–30 Sep, after the stemming, clean-text and fragment changes): plain keyword,
  field, node, restype, archive, paging and collection-order cases identical. Residual
  differences are catalogued in the review (themes A–H).
- Catalogue run (6 Oct): 269 cases, 143 identical; most of the rest are forms the Typesense-only
  group declines.
- Stemmer drift: 34 of 40 word-form pairs agree (irregular plurals differ).

## Headline findings still open

- Index freshness: nothing reaches the index between full reindexes (the on-save path is broken in
  more ways than three). Stale rows are not validated at read time, so a resource whose access or
  state changed can still be listed (existence, title, preview — not the file).
- Forms that return **everything** instead of declining: unresolved fixed-list values, a wildcard on
  an option, a negated phrase, a word of an option name.
- Forms that return **nothing**: year dropdown (`basicyear:`), quoted field searches (which the
  search bar itself generates), quoted phrases when stemming is on, leading wildcards, non-indexed
  date/numeric fields, date ranges whose end is not a real date.
- Typesense defaults the plugin never sets: prefix on the last word, candidate caps on wildcards,
  split/join retry.
- Relevance order differs from core (hit counts vs text match); keyword-less relevance is newest-first.
- CSV metadata export fails with a PHP TypeError for every group (null passed to a typed hook
  parameter).
- Core's own bugs (Core 1–10) and quirks, and which to copy, raise or ignore.

**2026-10-20:** every catalogue row marked; Q1.1 (b), Q0.4/Q11.1/Q11.2 (core changes made in ResourceSpace), Q11.6 (the plugin ships inside ResourceSpace 11.1) decided; deadline: finished before the end of 2026.

**2026-10-21:** the foundations questions decided (schema, client, state, tenancy, dates, text, testing); M0 waits only on the security set and the server list; scope for 11.1: nothing is cut.
