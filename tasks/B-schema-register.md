# Schema register

Every field the index holds or will hold, with its source, the package that needs it, the event that
keeps it fresh and the schema version it arrived in. The map for area B, as
[search/coverage.md](search/coverage.md) is for the search packages. B2's `scripts/schema.php
--register` will print this table from the registry so the two cannot drift; until then it is kept
by hand. *Version* 1 is the M1 baseline; later versions are assigned as packages land (Q4.2 decides
how they are batched). *Status* stays blank until the field's battery case is green.

Freshness abbreviations: **R** the resource events (register rows E1–E19) · **M** the membership
events (E30–E39) · **G** the grant events (E40–E49) · **rebuild** only the nightly rebuild (no hook
today) · **cfg** a configuration snapshot diff (E25–E27).

## 1. Resource-level fields (`<prefix>_resources`)

| Field | Type | Source | Flags | Needed by | Freshness | Version | Status |
|---|---|---|---|---|---|---|---|
| `id` / `ref` | int32 | `resource.ref` | id; sort | everything | R | 1 |  |
| `ref_s` | string | `resource.ref` as text | no stem | SA9 (a number as a word) | R | 1 |  |
| `title` | string | `$view_title_field` value | stem per Q2.14 | D4 `query_by`; D11 | R | 1 |  |
| `resource_type` | int32 | `resource.resource_type` | facet | D5, SF5 | R (E8 type change: gap until C9) | 1 |  |
| `archive` | int32 | `resource.archive` | facet, sort | D5, SF7, SG1 status sort | R (E6) | 1 |  |
| `access` | int32 | `resource.access` | | D6 | R (E7) | 1 |  |
| `created_by` | int32 | `resource.created_by` | optional (NULL absent) | D5, SF5, SF9 `cu` | R | 1 |  |
| `creation_date` | int64 | `resource.creation_date` epoch | | D5 day limit | R | 1 |  |
| `modified_date` | int64 | `resource.modified` epoch | sort | D9 modified sort | R (coarse: every `resource_log()`) | 1 |  |
| `date_field_sort` | string | `$date_field` value as `0:`/`1:`/`2:<value>` | sort, no stem | D9 date sort, Q7.1 (B) | R; cfg (`$date_field`) | 1 |  |
| `nodes` | int32[] | every `resource_node` row | facet | SC1, D7, SK1 | R (E9–E19 node writers: several gaps until C9) | 1 |  |
| `populated_field_ids` | int32[] | fields with any node, from all nodes | | SF6, SF10 | R | 1 |  |
| `date_year`, `date_month`, `date_day` | int32 | `$date_field` value components, absent when 0 | optional | SD2 | R; cfg | planned (Q3.13) |  |
| `hit_count`, `user_rating`, `rating` | int32 | `resource.hit_count` (the cron copy of `new_hit_count`), `resource.user_rating` (the star average), `resource.rating` (the legacy column from the rating field) | | SG2, D9; parts of the packed keys below | E27 (hit counts: rebuild only; ratings: the `modifyuserrating` hook and field edits) | planned (Q7.2 decided by the marks) |  |
| `ur_hits`, `hits_ur`, `date_ref`, `rating_ur` | int64, sort | packed keys built by B2: (user rating, hit count), (hit count, user rating), (`$date_field` as days, ref), (rating, user rating); fixed bit widths, "no value" = the minimum so NULL sorts as in MySQL in both directions | sort | popularity = `ur_hits, date_ref`; relevance with keywords = `_text_match, ur_hits, date_ref`; keyword-less relevance = `hits_ur, date_ref`; rating = `rating_ur`, then `_text_match` or hit count, then ref (D9, SG2) | R; E27; cfg (`$date_field`) | planned (Q7.2) |  |
| `has_image` | int32 | `resource.has_image` | | SF12, SF9 `pi` (SF15 parked 2026-10-20) | R (E13 preview creation) | planned (Q1.6) |  |
| `file_extension`, `file_size`, `width`, `height`, `orientation` | string / int64 / int32 / int32 / string | `resource.file_extension`, `file_size`, `resource_dimensions` | sort on extension | SF9, SG1 | R (E14 file replace; E15 alternatives) | planned (Q1.6, Q4.11) |  |
| `file_path` | string | `resource.file_path` | sort | SG1 | R | planned (Q7.2) |  |
| `lock_user`, `integrity_fail`, `has_file` | int32 / bool / bool | `resource.lock_user`, `integrity_fail`, `file_path` not empty | | SF13 | E16 lock (gap until C9), E17 integrity cron (rebuild) | planned (Q4.1) |  |
| `geo` | geopoint | `resource.geo_lat`, `geo_long` | | SF14 | R (set through `update_field`) | planned (Q4.1) |  |
| `colour_key` | string | `resource.colour_key` | no stem | SF15 | R (E13) | planned (Q4.1) |  |
| `related` | int32[] | `resource_related` both directions | | SF11 | E18 relate (gap until C9) | planned (Q4.1) |  |
| `title_sort`, `resource_type_order` | string / int32 | title lower-cased key; `resource_type.order_by` | sort | SG1 (resource type order = `resource_type_order, resource_type, ref`: three plain slots) | R; cfg (type order) | planned (Q7.2, decided by the marks) |  |
| `words` (type name, contributor username) | string[] | `resource_type.name`, `user.username` of `created_by` | in `query_by` | SA9 | R; cfg (type rename, E24) | planned (Q2.9, Q1.7) |  |
| `in_collections` | int32 | count of indexed memberships | | SF16 (if a negated join is impossible) | M | planned (Q4.7) |  |

## 2. Per-field representations (`field_<ref>_<suffix>`, dynamic patterns)

| Suffix | Applies to | Type | Value rule | Needed by | Freshness | Version | Status |
|---|---|---|---|---|---|---|---|
| `_s` | single-line text (types 0, 13) that is indexed and not numeric | string, stem per Q2.14 (the old schema also sets `facet`, unused — drop, Q1.2) | A5 `clean()` + cap (Q1.13) | SB1, D4 | R | 1 |  |
| `_text` | multi-line, large, TinyMCE (types 1, 5, 8), indexed | string, stem per Q2.14 | A5 (markup per Q1.4) | SB1, D4 | R | 1 |  |
| `_ss` | fixed lists (types 2, 3, 7, 9, 12): the option names | string[], facet, stemmed iff `$stemming` (already so in the old schema) | A5 `clean()` (BOM stripped, P33); translations per Q1.4 | SC3; D4 (option words as keywords, indexed fields only) | R (E20 node rename: gap until C9) | 1 |  |
| `_q` | dates (types 4, 6, 10, 14): year, year-month, full date; numeric single-line (`field_constraint = 1`): the plain value | string[] / string, never stem | A4 representations; numbers verbatim | SD1, SE1, SA9 (Q2.11) | R | 1 |  |
| `_f` | numeric single-line, `is_numeric` values only | float | the number | SE1, SG4 | R | 1 |  |
| `_ts` | full dates only | int64 epoch | A4 `timestamp()` | (none yet: candidate for removal, Q1.2) | R | 1 |  |
| `_range_start`, `_range_end` | every date type: the value at its own precision; type 14: the span | int64 | A4 | SD3 (b), SD4 | R | 1 |  |
| `_p` | partial-indexed text | string[] | A5 `fragments()` (every substring ≥ `$partial_index_min_word_length`) | SB6, SA5 (leading wildcards) | R; cfg (min length) | 1 |  |
| `_k` | every date type: the padded value's digits as an integer | int64 | A4 (raw digits, non-real dates kept) | SD3 (a) | R | planned (Q3.4) |  |
| `_c` | complete-indexed text: one token per value | string, no tokenisation | the whole value | SK3 | R | planned (Q1.12) |  |
| `_w` | text fields: the whole value when the cap applies to `_s`/`_text` | string | A5 without the cap | SK3 (wildcards past the cap) | R | planned (Q1.13) |  |
| `_sort` | fields core can order by (`$sort_fields`) | string / float | lower-cased key; numeric fields use `_f` | SG4 | R | planned (Q7.9) |  |

Selector rules (B2): a representation applies to a field only when the field's type and flags
match; `_s`, `_text`, `_ss` and `_p` need `keywords_index` / `partial_index` / `complete_index`
unless Q1.1 chooses (b); `_q`, `_f`, `_k`, `_range_*` apply to every date or numeric field
regardless of flags (Q3.16); inactive fields get no representations (P37, Q6.7); a field hidden by
permission is indexed but excluded from `query_by` and field filters at query time (catalogue K12).

## 3. Memberships and grants

| Collection | Field | Type | Source | Rule | Needed by | Freshness | Version | Status |
|---|---|---|---|---|---|---|---|---|
| memberships | `id` | `collection:resource` | `collection_resource` | de-duplicates duplicate rows | SF2, D8 | M | 1 |  |
| memberships | `resource_id` | string, reference → `resources.id` | | as the old schema names it; `async_reference`, `cascade_delete` are new | SF2, D8 | M, R (delete) | 1 |  |
| memberships | `collection_ref` | int64 | `collection_resource.collection` | | SF2, D8 | M | 1 |  |
| memberships | `sortorder` | int32 | `collection_resource.sortorder` | NULL → INT32_MIN (P18) | SF2 order | M (E32 reorder: gap until C9) | 1 |  |
| memberships | `date_added` | int64, sort | `collection_resource.date_added` | NULL → 0 (P18) | SF2 order | M | 1 |  |
| memberships | `collection_type` | int32 | `collection.type` | selection, upload, share-upload not indexed (P27); request, research, smart per Q5.7 | D8 (`j*`), SF2 | M (E36 type change: gap until C9) | 1 |  |
| grants | `id` | `resource:user:usergroup` | `resource_custom_access` | junk `(NULL, NULL)` rows skipped | D6 | G | 1 |  |
| grants | `resource_id` | string, reference → `resources.id` | | as the old schema names it; `cascade_delete` is new | D6 | G | 1 |  |
| grants | `user`, `usergroup` | int32, facet | | `−1` when unused (anonymous = 0 cannot match) | D6, D7 | G (E43–E47 writers without hooks: gap until C9) | 1 |  |
| grants | `access` | int32 | | | D6 | G | 1 |  |
| grants | `expires` | int64, range index | `user_expires` | 0 = never; checked at query time | D6 | G | 1 |  |

## 4. The field-type matrix (B1 and B4 batteries)

One row per ResourceSpace field type and flag combination; each must have a fixture field and a
golden document case. Representations assume the version-1 selector rules above.

| # | Field type / flag | Representations | Fixture field | Notes |
|---|---|---|---|---|
| FT1 | 0 single-line text, indexed | `_s` | 8 title | |
| FT2 | 0 single-line text, `field_constraint = 1` (numeric) | `_f` (numeric values), `_q`; `_s` for non-numeric values | 90 price (`abc` on resource 16) | the `$value == 1` bug |
| FT3 | 0 single-line text, `partial_index` | `_s`, `_p` | 51 originalfilename | |
| FT4 | 0 single-line text, `complete_index` | `_c` (planned), else `_s` | add | catalogue K9 |
| FT5 | 0 single-line text, not indexed | none (Q1.1 a) / `_w` (b) | add | catalogue B12 |
| FT6 | 1 multi-line text, indexed | `_text` | 18 caption | |
| FT7 | 1 multi-line text, not indexed | none / `_w` | 96 altnotes | catalogue B13 |
| FT8 | 5 large multi-line | `_text` | add | |
| FT9 | 8 formatted (TinyMCE) | `_text` after `strip_markup()` per Q1.4 | 92 notes | tag names, glued words |
| FT10 | 13 warning message | `_s` | add | |
| FT11 | 3 drop-down, indexed | `_ss`; nodes | 3 country | translated option on node 204 |
| FT12 | 3 drop-down, not indexed | nodes only | add | catalogue C8 resolves, C9 nothing |
| FT13 | 2 checkbox list | `_ss`; nodes | add | `$checkbox_and` forms |
| FT14 | 12 radio buttons | `_ss`; nodes | add | |
| FT15 | 7 category tree | `_ss`; nodes with stored ancestors | 73 subject | parent matches children |
| FT16 | 9 dynamic keywords | `_ss`; nodes | 1 keywords | multi-word options |
| FT17 | 10 date | `_q`, `_range_*`, `_ts` (full), `_k` (planned) | add | year-only and month-only values |
| FT18 | 4 date and optional time | as FT17; time ignored for `_q` | 12 date | catalogue D4, D23 |
| FT19 | 6 expiry date | as FT17 | 95 expiry | |
| FT20 | 14 date range | `_q` of both ends, `_range_*` = the span, `_k` of both | 91 eventdates | needs both endpoints |
| FT21 | any date or numeric type, not indexed | as its type (Q3.16) | add | catalogue D22, K11 |
| FT22 | any type, inactive | none (P37) | 94 oldfield | catalogue K6; negation Q6.7 |
| FT23 | any type, hidden by permission | as its type; excluded at query time | 93 secret | catalogue K12 |
| FT24 | a value over `$node_keyword_index_chars` | capped per Q1.13 | 18 on resource 6 | catalogue K10 |
| FT25 | a translated value `~en:A~fr:B` | per Q1.4 | node 204 | |
| FT26 | a value with a BOM / NBSP / word joiner | cleaned (P33) | add | catalogue C7 (a BOM option) |
| FT27 | a resource with no metadata at all | arrays present or optional (Q1.15) | add | 29 such resources on the test system |

## 5. Collection settings and versions

| Setting | Value | Reason |
|---|---|---|
| `token_separators` | `$config_separators` + `,` `:`, single-byte only | P31; curly quotes rejected |
| stemming | per field flag, iff `$stemming` — or none if Q2.14 pre-stems | A3, SA12 |
| `async_reference`, `cascade_delete` | on memberships and grants | B8; deletes cascade |
| metadata | `{generation, schema_version, plugin_version, started, complete}` | B8, B9 |
| `enable_nested_fields` | off | nothing nested |
| synonym set | `<prefix>_related` (if SA10 chooses synonyms) | C7 |

| Version | Contents | Packages | Rebuild |
|---|---|---|---|
| 1 | sections 1–3 as marked *1* | A3, B1–B9 | the M1 build |
| next… | assigned as planned rows land (Q4.2) | the row's *needed by* | one generation per version |
