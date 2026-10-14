# Chunk 7 — Sorting, relevance and result windows

Status: planning. Depends on: 0, 1 (sort fields), 8 (hit-count freshness). Feeds: 10.

## Scope

Mapping core's sort options to Typesense, the relevance model, which sorts fall back, the shapes of
`fetchrows` and their limits, deep paging, and the next/previous navigation.

## What we know

- Served sorts: relevance (keyword searches: `_text_match`; keyword-less: `ref desc`), date
  (`date_field_sort` text key `0:`/`1:`/`2:<value>` matching core's string sort of the cache column),
  modified (`modified, ref`), resource ID, collection order (3 slots). `!last` + relevance → ref
  order with core's upper-case-DESC test.
- Vetoed: rating, popularity (`$popularity_sort` is on by default, so the option is visible), colour,
  title, resource type, extension, file path, status, random, country, custom `field<n>`.
- Core's relevance = `score` (sum of hit counts of the matched keywords; for keyword-less searches
  the resource hit count), `user_rating`, `total_hit_count`, date field, ref. On the test system
  the first page of a node-only search has 0 of 48 in common with core; sorting by `hit_count`,
  `date_field_sort`, `ref` reproduces 387 of core's top 400 (option B). Core updates hit counts by
  cron, with no hook.
- Windows: `fetchrows` `-1`, `N`, `[offset, limit]`, `[0,0]` (count) all handled; windows walked with
  `multi_search` (≤ 50 × 250 per request), de-duplicated, hydrated in ~1,000-ref batches (MySQL's
  65,535-placeholder cap at ~32k refs); int-`N` results zero-padded like core; cost 25k rows 1.5–4 s,
  50k 10 s, whole index 20–37 s; `$typesense_search_max_rows` 25,000 — above it the search falls
  back to core.
- Next/previous on `view.php` fetches all rows (`-1`): above 25,000 results it falls back to MySQL
  while the grid stays on Typesense → different order under relevance.
- The search page passes `[offset − colcount, n]` when collections appear above results, so page 2+
  is misaligned.
- Bare numbers with `$config_search_for_number` off: core puts the exact ref first; `_eval(ref:=N)`
  reproduces this (verified) but is not implemented (the search vetoes).
- Typesense: string sort is case- and accent-insensitive; empty = missing; missing sorts last
  ascending, first descending; no `missing_values`; `_eval` works; 3 sort fields max; no random sort.
- Core bug: ordering by a metadata field that is not a resource-table join column returns nothing
  (Core 8).

## Candidate work items

- The relevance decision (Q7.1) and, if B, `hit_count`/`user_rating` in the schema.
- Sort keys for the sorts chosen in Q7.2.
- Exact-ref-first via `_eval`.
- Next/previous and the page-offset misalignment (Q7.3, Q7.5).

## Questions to answer before coding

### Q7.1 — The relevance model (open decision D2)
(A) accept: text match for keyword searches, newest-first for keyword-less — different from core,
documented; (B) index `hit_count` and `user_rating` and sort keyword-less searches by
`hit_count, date_field_sort, ref` (387/400 agreement; fresh only after a rebuild) — keyword searches
still cannot reproduce core's per-keyword score; (C) veto keyword-less relevance to core meanwhile.
Also the underlying question from the review (G): is a different relevance order acceptable to the
users at all?

### Q7.2 — Which further sorts to serve
Each needs a sortable field: title (a case-insensitive key), status (`archive` is already sortable —
trivial), resource type (`rty.order_by`), extension and file path, rating and popularity (with
freshness caveats), custom `field<n>` (numeric and dot-notation rules), country. Random can never be
served. Which, in what order?

### Q7.3 — The row limit and next/previous
Keep 25,000? Lower? For next/previous: make the plugin serve `-1` fully (20–37 s on the whole index —
too slow), force `view.php` navigation to core always (consistent order only if the grid is too),
or change core's next/previous to a windowed query (core change, Q0.4). Recommendation: a core
change; it is the only way the grid and navigation agree.

### Q7.4 — Exact-ref-first for bare numbers
Implement with `_eval(ref:=N):desc` as the first sort key. Confirm.

### Q7.5 — Page offset with collections above results
Fix in the plugin (derive the page from the raw offset, as the plan notes) or in core (pass an
aligned window)?

### Q7.6 — Row shape
Confirm the remaining shape differences are acceptable: the `score` column, the suggestion string on
zero results (integer `fetchrows` only — the search page passes an array, so nothing is lost there),
`[0,0]` totals.

### Q7.7 — Deep paging cost
Walk cost grows faster than linear (deep pages re-rank). Keyset on `ref` works only for numeric sorts.
Acceptable as is, or should the plugin veto a window that starts beyond some offset?

### Q7.8 — Sort direction quirks
`!last`'s upper-case-DESC test is replicated. Are other case-sensitivity quirks (API lower-case
`desc`) to be replicated or raised with core?

### Q7.9 — Ordering by a metadata field that is not a join column
`order_by=field<n>` works in core only for the fields in the resource-table joins (`$sort_fields`):
field 12 orders (G25), a numeric field would order with `+0` (G26), the title field by number orders
as text (G27), and any other field makes core write an invalid column and return **nothing** (G43,
Core 8). Serve only the join-column fields (parity, including nothing for the rest), or serve any
field from a per-field sort key (an improvement that returns rows where core returns none)?

## Done when

Catalogue group G has a decided behaviour per row; each served sort matches core's order on the
harness ties fixture and live; the relevance decision is documented in the behaviour spec.
