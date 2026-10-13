# Chunk 3 — Field, date, numeric and node searches

Status: planning. Depends on: 0, 1. Feeds: 10.

## Scope

`field:value` in every form the pages and API produce: text fields, fixed-list fields (resolved and
unresolved), quoted field searches, the date dropdowns (`basicyear:` …), date values and ranges,
numeric values and `numrange`, non-indexed fields, negation; and `@@` node tokens with their
Typesense limits.

## What we know

- Text `field:value` → bare-colon word-contains on `_s`/`_text`; identical to core.
- Fixed-list `field:value` is resolved by core into `$node_bucket` **only when the value is an exact
  option name**. When it is not (a word of an option, a wildcard, a category-tree child by name,
  dynamic-keyword text, an option whose name starts with a BOM), the plugin still skips the token
  as "already in the bucket" and **returns everything** (`usagerights:social` 148 / 100,862). The
  worst class of divergence.
- Quoted field searches (`"title:launch party"`, `"usagerights:Social media"`) go into `q` as a
  phrase → 0. The search bar itself generates them when a user clicks a metadata value with a
  space, and generates `subject:Birds` for a tree child (unresolved → everything).
- `basicyear:` / `basicmonth:` / `basicday:` (simple and advanced date dropdowns) are not recognised →
  0 (`basicyear:2024` 8,838 / 0).
- Dates: `date:2024` now exact on `_q` representations (fixed); ranges are an interval overlap on
  `_range_start/_end` while core string-compares raw values (year-only `2020-00-00` never matches a
  range starting after 1 Jan: +106); a range end that is not a real date (the form pads `-31`, EDTF
  `-99`) is dropped by the plugin but works lexically in core (`2024-02-31`: 822 / 21,151); one value
  on a date-range field returns nothing in core (bug); a date-range field exact search has its SQL
  parameters reversed in core (bug).
- Numeric: `numrange<min>|<max>` → `_f` range, single bound = exact (core); plain `field:42` on a
  numeric field filtered `_s` → 0 (a `_q` mapping is staged); `numrange` on a text field compares
  titles as numbers in core.
- Non-indexed fields: core works on node names for date, numeric and value searches; the plugin has
  no attributes for them → 0, no fallback.
- Partial-indexed text fields filter on `_p`; inactive fields veto (staged); non-viewable fields
  veto; negative field searches veto (core degrades them to loose words).
- Nodes: parity by construction — core resolves every `@@` token before the hook; `nodes:!=[a,b]`
  is any-of like core's `NOT EXISTS`; AND across buckets; a GET query string is capped at 4,000
  bytes and `filter_by` at 100 operations, so the first window goes via POST `multi_search` with a
  `$typesense_search_filter_max_ops` guard: 1,000-id OR buckets served, AND chains to 40 nodes;
  an over-limit join expression ANDed with others returns 0 silently. Untested: category-tree
  parent/child (no tree field in the test data), `$category_tree_search_use_and_logic`, a node
  filter on a non-indexed field. Keyword-less node searches order differently (chunk 7).

## Candidate work items

- A single rule: any `field:value` the plugin cannot fully resolve → veto. Covers unresolved fixed-list
  values, wildcards on options, tree children by name, dynamic-keyword text, BOM-prefixed names.
- Recognise quoted field searches and the search bar's rebuilt forms; recognise `basic*:`.
- Date-range semantics per Q3.4, including non-real end dates.
- Numeric plain value → `_q`; `numrange` on text fields → decide.
- A fixture with a category tree, a dynamic keyword field, a date-range field and a numeric field
  (chunk 10).

## Questions to answer before coding

### Q3.1 — Confirm the "decline rather than return everything" rule
Every unresolved field value becomes a veto. Any case where serving a best-effort match is preferred
(e.g. a word of an option name → word-contains on `_ss`)?

### Q3.2 — Quoted field searches and search-bar text
These are reached by clicking a metadata value, so they are common. Implement: strip the quotes,
scope the phrase to the field (text) or resolve the option (fixed list)? Or veto?

### Q3.3 — `basicyear:` / `basicmonth:` / `basicday:`
Implement against the date field's representations (year, month, day parts are indexed in `_q`).
Confirm, and confirm `$date_field` is the target when the dropdowns are used.

### Q3.4 — Date-range semantics (open decision C)
(a) Replicate core's string comparison exactly — index a fixed-width sortable key (e.g. the padded
value's digits as an integer, so `2024-02-31` and `2020-00-00` compare as core's strings do) and
filter numerically; (b) keep interval overlap (more correct; +106 on the test system, and the
behaviour doc describes it); (c) overlap but document. Either way the non-real end date must be
handled. Recommendation: (a) if the goal is parity (Q0.2), since the advanced form produces these
ends routinely.

### Q3.5 — Date-range fields (start/end pairs)
Core is buggy here (exact value → nothing; parameters reversed). Serve correctly, replicate the bug,
or veto? (Links Q11.3.)

### Q3.6 — Field-specific searches on non-indexed fields
Options: index every field's values (Q1.1); veto `field:value` when the field has no index flag
(the lookup knows); accept 0 results. The test system has one such date field in use.

### Q3.7 — Numeric fields
Confirm plain `field:42` → `_q` exact (staged). `numrange` on a non-numeric text field: core compares
titles as numbers — veto or replicate? Open-ended comparisons (`>=` without an upper bound) do not
exist in core: add as an improvement or not?

### Q3.8 — Negation of field terms, dates, numbers and nodes inside OR groups
Core cannot do any of these; the plugin vetoes negative field searches. Keep (consistent) or go
beyond core?

### Q3.9 — Node limits and toggles
`$category_tree_search_use_and_logic` is implemented but untested — get a tree fixture before
relying on it. AND chains above 40 nodes veto to core (core itself fails at 58 joins): acceptable?
Raise the server's `--filter-by-max-ops` on the target and the plugin's guard with it?

### Q3.10 — Wildcard candidate caps on field searches
`field:word*` is capped at 4 completions by default (`title:zeb*` 40 / 4). Raise to what, or
exhaustive?

### Q3.11 — Field name lookup
Core matches short names exactly (case?); unknown names become two words (replicated). Confirm
case handling and whether a field name that collides with a special keyword (`date`) needs care.

## Done when

Catalogue groups B, C, D and E each have a decided behaviour per row, a harness case, and identical
live counts for the "copy core" rows; the "returns everything" class is gone from the review.
