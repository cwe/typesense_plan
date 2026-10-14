# SG — Parameters beyond the pipeline (catalogue group G)

Group G is mostly the plumbing: resource types, workflow states and the day limit are D5, the
served sorts are D9, windows are D10, shapes are D11, the request modes are D1 (see
[coverage.md](coverage.md) for the row-by-row ownership). What is left are the **sorts the base
mapping declines** — each needs a sortable field and a rebuild, so each is its own package. Order
and inclusion are Q7.2; all milestone M7 unless promoted.

The rule they share (D9): a sort the plugin cannot reproduce vetoes; it never silently sorts by
relevance (harness O19 shows what Typesense-only mode does with a vetoed sort).

### SG1 — Text-key sorts: title, status, resource type, extension, file path
- **Rows:** G16 G17 G19 G20 G21 G24
- **Builds:** sort keys `title_sort` (case-insensitive, from `$view_title_field`), `archive`
  (already sortable — status is trivial), `resource_type_order` (`rty.order_by`, then type,
  then ref), `file_extension`, `file_path`; the mapping of each `order_by` fragment.
- **Size:** M · **Depends on:** D9, A3, B2 · **Milestone:** M7
- **Core:** title = `$view_title_field, ref`; status = `archive, ref`; resourcetype = the type's
  `order_by`, then type, then ref; extension = `file_extension, ref`; file_path = `file_path, ref`;
  country = field 3's value when that field exists, otherwise relevance (G24 — absent on the test
  system, so parity is "same as G11").
- **Serve:** one sort slot each plus `ref` (three slots maximum); Typesense string sort is case-
  and accent-insensitive, an empty string is "missing", missing sorts last ascending and first
  descending with no `missing_values` option — core's `ORDER BY` puts NULL and empty first
  ascending: a `0:`/`1:`/`2:<value>` prefix key as `date_field_sort` uses (P19) reproduces it.
- **Fixture:** a ties fixture (equal titles, empty titles, NULL extensions).
- **Battery:** `harness/ab/SG1_text_sorts.php` — each row both directions on the ties fixture,
  then live first-five refs against the catalogue.
- **Blocking questions:** Q7.2, Q7.8

### SG2 — Rating and popularity
- **Rows:** G12 G13 · cross: G11 (D9, Q7.1)
- **Builds:** `user_rating`, `rating`, `hit_count` on the resource; the two mappings.
- **Size:** S · **Depends on:** D9, B2, C7 (E27: the counts refresh only at a rebuild) · **Milestone:** M7
- **Core:** popularity = `user_rating, total_hit_count, date, ref`; rating = `r.rating,
  user_rating, score, ref`; `$popularity_sort` is on by default so the option is visible; core
  updates hit counts by cron with no hook, so the index can only be as fresh as the last rebuild
  (Q8.9).
- **Serve:** the sort keys with the freshness caveat documented; the same fields answer Q7.1 (B)
  for keyword-less relevance (387 of core's top 400 reproduced).
- **Battery:** `harness/ab/SG2_rating_popularity.php` — G12, G13 on a fixture with known counts;
  live first-five refs.
- **Blocking questions:** Q7.1, Q7.2, Q8.9

### SG3 — Colour
- **Rows:** G15
- **Builds:** `has_image, blue, green, red` on the resource; the mapping.
- **Size:** S · **Depends on:** D9, B2 · **Milestone:** M7
- **Core:** `has_image, blue, green, red, date, ref` (six keys — Typesense allows three sort
  fields, so this needs a composed key or stays declined).
- **Battery:** `harness/ab/SG3_colour_sort.php` — G15.
- **Blocking questions:** Q7.2

### SG4 — Ordering by a metadata field (`field<n>`)
- **Rows:** G26 G27 G43 · cross: G25 (D9: field 12 is the date field)
- **Builds:** `order_by=field<n>` for fields other than `$date_field`: a per-field sort key, or
  the veto.
- **Size:** M · **Depends on:** D9, B1, B2 · **Milestone:** M7
- **Core:** `field<n>` sorts by that field's value then ref **only when the field is a
  resource-table join column** (`$sort_fields`): field 12 works (G25 — the date field, already
  D9's mapping); a numeric field adds `+0` for numeric order (G26); the title field by number is a
  text order (G27); a field that is not a join column makes `check_order_by_in_table_joins()`
  swap to resourceid and the field branch then writes "resourceid DESC" as a column name — an SQL
  error, so **nothing** (G43, Core 8).
- **Serve:** sort keys only for the join-column fields (parity), or for any field (an
  improvement — but then G43 returns rows where core returns nothing); numeric fields sort on
  `_f`; text on a `_sort` key; Q7.9 decides.
- **Battery:** `harness/ab/SG4_field_sorts.php` — G25–G27, G43 on the fixture's join-column and
  non-join-column fields.
- **Blocking questions:** Q7.2
- **Open questions:** Q7.9

### SG5 — Random
- **Rows:** G23
- **Builds:** nothing: the veto and its doc line.
- **Size:** S · **Depends on:** D9 · **Milestone:** M2
- **Core:** `RAND()`.
- **Serve:** never (Typesense has no random sort); the battery asserts the veto.
- **Battery:** in `D9`'s sort battery.
- **Blocking questions:** —
