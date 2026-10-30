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
- Typesense (docs 30.0, read 16 October): three sort fields, final ties by insertion order,
  `_text_match` appended when fewer than three are given; string sort needs `sort: true` and a
  separate index; missing numeric values always last, missing strings highest, and
  `field(missing_values: first|last)` overrides both (an earlier live note said the option did not
  exist — verify on 30.2); `_eval(<filter>)` for conditional ordering; `_rand([seed])` exists but can
  never match core's `RAND()`; string sort was case- and accent-insensitive in the live tests.
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
**Direction from the acceptance rule (Q10.3, 2026-10-16):** a keyword-less search ordered by
relevance has no text string, so its order is exact — hit count, user rating, total hits, date field,
ref, which is five keys where Typesense allows three: a composite sort key built at index time
(option B, extended). Hit counts move by core's cron with no hook, so the order is exact as of the
last rebuild. Still to say: is that staleness acceptable, or does keyword-less relevance decline?

**Catalogue marks (2026-10-20):** G11 (relevance) is *check* — Chris checks core's relevance order, the
keyword and the keyword-less case, before the mark is final; the direction above stands meanwhile.

**Decided (2026-10-30):** G11 is *must* with the Q7.1 difference accepted and documented: keyword searches order by the text-match score where core uses the matched-node hit-count sum; keyword-less relevance is exact.

**Decided 2026-10-20: option (B).** The hit-count cluster is in the version-1 schema; a keyword-less search
ordered by relevance is served exactly (hit count, user rating, date field, ref via the packed keys) and
kept in step with core by a new hook at the end of `copy_hitcount_to_live()` — core copies hit counts
live at most once a day — plus the existing `modifyuserrating` hook and the edit hook. **Divergence,
documented:** a keyword search ordered by relevance differs from core — core's score is the per-search
sum of the matched nodes' hit counts, which Typesense cannot reproduce; the plugin orders by its text
match, then the same keys. The one accepted difference under Q10.3, listed in the harness and in the
user documentation.

### Q7.2 — Which further sorts to serve
Each needs a sortable field: title (a case-insensitive key), status (`archive` is already sortable —
trivial), resource type (`rty.order_by`), extension and file path, rating and popularity (with
freshness caveats), custom `field<n>` (numeric and dot-notation rules), country. Random can never be
served. Which, in what order?
**Catalogue marks (2026-10-16):** every sort is *must* except colour (G15), random (G23) and country
(G24): popularity, rating, title, file path, resource type, extension, status and the metadata-field
sorts are required (SG1, SG2, SG4); SG3 is parked; SG5 stays a decline. Core's own facts for the two
count-based sorts: `hit_count` is copied from `new_hit_count` by the maintenance cron (views and
downloads bump `new_hit_count` at once), `user_rating` is the star-rating average written by
`set_user_rating()` (`$user_rating` is off by default), and `r.rating` is a legacy column filled
from a rating metadata field when `$orderbyrating` is on — on the test system both are NULL, so the
rating sort there is ref order. Multi-key orders exceed Typesense's three sort slots and need a
composite key built at index time. **Noted 2026-10-16:** four packed `int64` keys cover all
three over-length orders — `ur_hits`, `hits_ur`, `date_ref`, `rating_ur` (SG2 has the table); "no
value" encoded as the minimum reproduces MySQL's NULL placement in both directions.

**Catalogue marks (2026-10-20), amended:** date (G14), resource id (G18), modified (G22), unknown (G28)
and the direction rows (G29, G30) stay *must*; popularity (G12), rating (G13), title (G16), file path
(G17), resource type (G19), extension (G20), status (G21) and a date field by number (G25) are *check*,
so SG1 and SG2 wait on Chris's check of core — what `rating`, `user_rating` and `hit_count` do there
in particular; colour, random and country stay *no*. The packed keys remain the design for SG2.

**Decided (2026-10-30):** G12, G13, G16, G19, G20, G21 and G25 are *must*; G17 (file path) is *no* — an unexercised admin-only key. G13 carries a second accepted difference: after `r.rating` and `user_rating`, the text-match score replaces core's hit-count tie-breaker for keyword searches, tested apart from G11. G19 indexes the type's configured order, kept in sync, sorted with `missing_values` first ascending and last descending. SG1 (without G17) and SG2 are required.

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
**Catalogue marks (2026-10-16):** G26 and G43 (a field that is not a join column) are *must* — core
returns nothing, so declining satisfies it; G27 (the title field by number) is *must* and needs a
per-field sort key.

**Catalogue marks (2026-10-20):** G26 and G27 are *check*, so SG4 waits on the check; G43 keeps its mark
until Chris recalls why it sat apart — it is the Core 8 row, and whether the bug is copied, fixed or
declined is Q11.3, separate from the mark.

**Decided (2026-10-30):** G27 is *must* (the title field by number, text order); G26 and G43 stay bug rows (Core 8); SG4 is required.

## Done when

Catalogue group G has a decided behaviour per row; each served sort matches core's order on the
harness ties fixture and live; the relevance decision is documented in the behaviour spec.
