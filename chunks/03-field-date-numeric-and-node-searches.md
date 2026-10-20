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

**Catalogue marks (2026-10-20):** B3 (`"title:henry moore"`) and B4 (`title:"henry moore"`) are *check* —
core's handling of both quoted forms is confirmed before the marks are final; SB2 keeps the rows.

### Q3.3 — `basicyear:` / `basicmonth:` / `basicday:`
Implement against the date field's representations (year, month, day parts are indexed in `_q`).
Confirm, and confirm `$date_field` is the target when the dropdowns are used.
**Catalogue marks (2026-10-16):** D10–D15 are *must* — the dropdowns and the legacy `nnnn|05` form
are served; Q3.13 chooses the mechanism.

**Catalogue marks (2026-10-20):** H19 and H20 (the simple-search year with a keyword and with a node) are *must*
— SD2 must compose with both.

### Q3.4 — Date-range semantics (open decision C)
(a) Replicate core's string comparison exactly — index a fixed-width sortable key (e.g. the padded
value's digits as an integer, so `2024-02-31` and `2020-00-00` compare as core's strings do) and
filter numerically; (b) keep interval overlap (more correct; +106 on the test system, and the
behaviour doc describes it); (c) overlap but document. Either way the non-real end date must be
handled. Recommendation: (a) if the goal is parity (Q0.2), since the advanced form produces these
ends routinely.
**Direction from the acceptance rule (Q10.3, 2026-10-16):** a served range must give core's set, so
the string comparison is reproduced (the sortable key) or ranges with non-real ends are declined;
interval overlap as a served difference is out.

**Catalogue marks (2026-10-20):** H35 (a date range with a node and a keyword), J5 (a start year only) and J6
(months only) are *must*; J13 (a value and a range on one field) is *check*.

### Q3.5 — Date-range fields (start/end pairs)
Core is buggy here (exact value → nothing; parameters reversed). Serve correctly, replicate the bug,
or veto? (Links Q11.3.)
**Catalogue marks (2026-10-16):** D24 and D25 are *check* — Chris verifies core's behaviour on a
date-range field before deciding.

### Q3.6 — Field-specific searches on non-indexed fields
Options: index every field's values (Q1.1); veto `field:value` when the field has no index flag
(the lookup knows); accept 0 results. The test system has one such date field in use.

**Catalogue marks (2026-10-20):** K11 (a number range on a non-indexed numeric field) is *check*; no such field
exists on the test system.

### Q3.7 — Numeric fields
Confirm plain `field:42` → `_q` exact (staged). `numrange` on a non-numeric text field: core compares
titles as numbers — veto or replicate? Open-ended comparisons (`>=` without an upper bound) do not
exist in core: add as an improvement or not?
**Catalogue marks (2026-10-16):** E1–E5 are *must* and worked with the previous code (the `_f`
range and single-bound forms, live-validated on field 170); E6 (a plain number in a numeric field,
which core treats as a keyword in the field) and E7 (`numrange` on a text field) are *check*.

### Q3.8 — Negation of field terms, dates, numbers and nodes inside OR groups
Core cannot do any of these; the plugin vetoes negative field searches. Keep (consistent) or go
beyond core?

**Catalogue marks (2026-10-20):** B17 (`title:-sculpture`) is *check* — whether core really excludes nothing is
confirmed before the mark is final (a bug would fall under Q11.3); SB5 keeps the row.

### Q3.9 — Node limits and toggles
`$category_tree_search_use_and_logic` is implemented but untested — get a tree fixture before
relying on it. AND chains above 40 nodes veto to core (core itself fails at 58 joins): acceptable?
Raise the server's `--filter-by-max-ops` on the target and the plugin's guard with it?

### Q3.10 — Wildcard candidate caps on field searches
`field:word*` is capped at 4 completions by default (`title:zeb*` 40 / 4). Raise to what, or
exhaustive?

**Catalogue marks (2026-10-20):** H5 (a field wildcard and a node) is *must*.

### Q3.11 — Field name lookup
Core matches short names exactly (case?); unknown names become two words (replicated). Confirm
case handling and whether a field name that collides with a special keyword (`date`) needs care.

### Q3.12 — Serve unresolved option text through the option names (`_ss`)
A value core does not resolve to an option becomes a keyword search within the field (C4
`usagerights:social` 148; C12 `artistname:moore` 2,713; C5 `soc*`; C7 a BOM-prefixed name), and
nothing on a fixed-list field that is not flagged for indexing (C9: 0). The bare-colon word match on
a `field_<ref>_ss` array of option names (tokenised and stemmed like text) reproduces this, and a
missing `_ss` on a non-indexed field matches nothing, as core. Confirm this replaces the veto of
Q3.1 for SC3, and that `_ss` is stored for every indexed fixed-list field (Q1.1).

**Catalogue marks (2026-10-20):** C4 (one word of an option), C5 (a wildcard on an option), C7 (the AND of two
options) and C12 (one word of a dynamic keyword) are *check* — core's keyword-in-field fallback and the C7
result are confirmed before the marks are final; SC3 keeps the rows. C23 (an option name and a node of the same
field, SC2) is *check* for the same reason: its 0 in core is to be confirmed.

### Q3.13 — `basicyear` / `basicmonth` / `basicday`: integer components or `_q` tokens?
The dropdowns address `$date_field` only, with `LIKE "____-05-__%"` forms that `_q`'s year /
year-month / full-date tokens cannot express. Options: three integer fields on the resource for
`$date_field` (`date_year`, `date_month`, `date_day`), filtered with `:=` and ANDed; or month-only
and day-only tokens in `_q` for every date field. Recommendation: the integers — exact, cheap, and
the legacy `nnnn|05` form maps to the same fields. Also: `basicday` is handled by core whether or
not `$searchbyday` is set — replicate that.

### Q3.14 — Two values and `;` on one date field
`date:2024, date:2025` is two joins ANDed — only resources holding both years (D26: 0); `date:2024;2025`
is `LIKE "2024;2025%"`: nothing (D27). Replicate both (two ANDed `_q` filters; an impossible filter
or a veto for the `;` form)? The SA7 veto of `;` must not treat a date field as an OR group.
**Catalogue marks (2026-10-16):** D26 (two values on one field) is *must*; D27 (`;` on a date) is
*check*.

### Q3.15 — `$ignore_filters`
It reaches the hook and core uses it only to skip the date, `numrange` and fixed-list conversion of
field terms (G41). Read it the same way in the parse (so `date:2024` with the flag is plain words),
or veto when it is set? Which callers set it?
**Catalogue marks (2026-10-16):** G41 is *must* — read the flag as core does.

**Catalogue marks (2026-10-20):** G41 is *check* — which callers set `$ignore_filters` and what core does
with it are confirmed before the mark is final.

### Q3.16 — Non-indexed date and numeric fields as a slice of Q1.1
Date and number terms join nodes in core, so the index flag never matters for them (D22, D23, K11).
Index `_q` / `_f` / `_k` for every date and numeric field regardless of `keywords_index`, whatever
Q1.1 decides for text? Recommendation: yes (SD5).
**Catalogue marks (2026-10-16):** D22 and D23 (a date field not flagged for indexing) are *check*;
if required, this is the slice that serves them.

### Q3.17 — The exact rules of the field-name match
Core compares the short name case-insensitively (I11) and whole (a slash is allowed, B9); a
trailing or leading colon gives an empty or unknown name (I6, I7); `date` is a real field short name
on the test system. Confirm the lookup: exact, case-insensitive, no trimming of other characters,
`basic*` reserved (Q3.13), and an unknown name → the colon becomes a space.

### Q3.18 — Is a backtick-quoted multi-word bare-colon value a phrase or an AND?
SB2 and SB4 need `"title:henry moore"` and `exhibitiontitle:fabric-ation` as phrases confined to the
field (core's rule). Verify live whether `field_8_s:\`henry moore\`` requires adjacency; if it is an
AND of words, choose between an accepted difference and a veto for multi-word field phrases.

**Catalogue marks (2026-10-20):** B3 is *check* (see Q3.2).

## Done when

Catalogue groups B, C, D and E each have a decided behaviour per row, a harness case, and identical
live counts for the "copy core" rows; the "returns everything" class is gone from the review.
