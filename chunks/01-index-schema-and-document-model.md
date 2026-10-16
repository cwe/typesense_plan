# Chunk 1 — Index schema and document model

Status: planning. Depends on: 0. Feeds: 2, 3, 4, 7, 8.

## Scope

What the Typesense index holds for each resource, membership and grant; how values are represented
per field type; cleaning, stemming and tokenisation at index time; extra fields needed by sorts and
special searches; how the schema changes over time.

## What we know

- Three collections per prefix (resources, memberships, grants); dynamic field patterns
  `field_.*_{s,ss,text,q,f,ts,range_start,range_end,p}`; `nodes[]` and `populated_field_ids[]`
  from **all** of a resource's nodes (with a `0` sentinel so the arrays always exist); `field_*`
  values only from fields flagged `keywords_index` / `partial_index` / `complete_index`.
- Per type: single-line text → `_s` (numeric-constrained → `_f` + `_q`); multi-line → `_text`;
  fixed lists → `_ss`; dates → `_q` (representations), `_ts` (full dates only), `_range_start/_end`
  (year, month, day precision; date-range fields: the span); partial-indexed text → `_p` fragments.
- Text fields stem iff `$stemming` (`_q` never); `token_separators` = `$config_separators` plus `,`
  and `:` (single-byte only — Typesense rejects curly quotes); invisible characters stripped (BOM,
  NBSP, word joiner: 6,548 documents held a BOM-prefixed option).
- Fragments vs infix (measured on 114,596 filenames): fragments give exact RS counts, +108 MB, +15 s
  import, 2–3 ms queries; infix is +3 MB but wrong for multi-word searches. Fragments chosen.
- Core reads **non-indexed** fields for wildcard and full-text searches (MySQL `MATCH … AGAINST` over
  every node), and for field-specific date/numeric/value searches on node names. The plugin indexes
  only flagged fields, so those searches return 0 without fallback (`datephototaken:2024` 3,676 / 0;
  `sculpt*` 49,610 / 51,007 where field 86 is not keyword-indexed).
- Core indexes the first `$node_keyword_index_chars` (500) characters; the plugin indexes whole
  values. Raw HTML is indexed (tag names searchable, a word after a tag glued to it); translated
  values `~en:A~fr:B` are glued (`afr`). Neither occurs in the YSP data.
- Not indexed today: `hit_count`, `user_rating` (relevance/popularity/rating sorts), `has_image`,
  file size/extension/dimensions, lock/integrity/file-path flags, geo, colour key, related refs,
  resource-type names, a sortable title. Each blocks a sort or a special search.
- Typesense 30.2: rejects a numeric string for an int field; schema PATCH adds and drops fields on a
  live collection; a reference may name an alias but binds to the real collection; per-collection
  synonyms API is gone (synonym sets instead); string sort is case- and accent-insensitive; an empty
  string counts as missing; missing values sort last ascending, first descending; no
  `missing_values` sort option; `_eval` sorts work; 3 sort fields max.
- Memory: ~800 MB resident for the test index with fragments.
- Indexer bug fixed: `_f` was only written when the value equalled 1; now `field_constraint == 1 &&
  is_numeric`. `created_by` is optional (NULL for CLI-created resources). Inactive fields are
  excluded (staged).

## Candidate work items

- Decide the field inventory and representation per type (below), then write the schema as a single
  function with a version number.
- Generation collections behind aliases (from the sync plan) as the only way a schema change reaches
  production; `PATCH` upgrades only for additive changes.
- Index-time value normalisation matching core (HTML, translations, length cap) where decided.
- Extra fields for the sorts and specials chosen in chunks 4 and 7.

## Questions to answer before coding

### Q1.1 — Index only flagged fields, or every field's values?
Core's wildcard, full-text and field-specific date/numeric searches reach non-indexed fields. Options:
(a) keep flagged-only and veto the forms that need more (field:value on a non-indexed field is
detectable; a bare wildcard is not); (b) index every active field's values, searchable or not
depending on flags (cost: size, and `query_by` must still exclude non-indexed fields for plain
keyword searches); (c) accept the difference. The size cost is unmeasured.
**Direction from the acceptance rule (Q10.3, 2026-10-16):** wildcard and full-text forms over
non-indexed fields must give core's set, so either every field's values are indexed (b) or those
forms decline when a visible non-indexed text field exists; serving them approximately is out.

### Q1.2 — Which representations per field type?
Keep the current set? Specific points: `_q` for text fields is unused (the bare-colon filter on
`_s`/`_text` does the job) — drop it; add a sortable key per field that core can sort by (title,
custom `field<n>` with numeric and dot-notation rules, status) — which fields; keep `_ts` when
`_range_start` covers every case.

### Q1.3 — Fragments: keep, and should they be free-text searchable?
Fragments are in `query_by` for partial-indexed fields, so a word held only in a fragment is
findable. Catalogue E14 says fragments are *not* free-text searchable — reconcile (one of the two is
out of date) and confirm the intended behaviour. Also confirm the memory cost is acceptable on the
target server.

### Q1.4 — Value cleaning at index time
(a) Strip HTML tags and entities before indexing (core strips? verify) — yes/no; (b) translated
values: index every language, only the default, or per-language fields; (c) cap at
`$node_keyword_index_chars` like core, or index whole values (more matches than core); (d) keep the
invisible-character stripping. Note the test data has none of (a)–(c), so decide on principle.

### Q1.5 — Stemming and tokenisation
Keep stemming gated on `$stemming` and the separator set. Stemmer drift (34/40 pairs agree):
accept, or add custom stems / synonym sets for the irregular plurals seen? Accent folding: Typesense
folds; core relies on the MySQL collation — treat as equivalent?

### Q1.6 — Which extra fields go in now?
Candidates and what they unlock: `hit_count` (+`user_rating`) → relevance/popularity/rating sorts
(fresh only via rebuild); `has_image` → `!images`, `!nopreview`; file size/extension/width/height →
`!properties`, extension sort; `lock_user`, `integrity_fail`, `no_file`, `file_path` flags →
`!locked`, `!integrityfail`, `!noningested`; `geo_lat/long` → `!geo`; `colour_key`/RGB → `!colour`,
`!colourkey`, `!rgb`-adjacent; related refs → `!related`; `rty.order_by` → resource-type sort; a
sortable title → title sort. Add as one batch (one rebuild) or per chunk?

### Q1.7 — Resource-type names and contributor usernames as searchable words
Core matches "Photo" to every resource of that type, and usernames when `$index_contributed_by` is
on. Index a `type_name` (and `contributor`) token field, or veto searches containing a type name?

### Q1.8 — Schema evolution
Confirm generation collections + aliases as the mechanism (prior decision P7); what triggers a full
rebuild (field config change, `$stemming` change, plugin schema version change); how the schema
version is recorded (alias name suffix, a metadata document); whether `PATCH` upgrades remain for
additive changes between rebuilds.

### Q1.9 — Typesense version and features
Pin to 30.x and use synonym sets, `_eval` sorts, `multi_search`, reference joins, `cascade_delete`,
aliases. Any reason to support an older server?

### Q1.10 — Document identity
Resources: `id = ref`; memberships: `collection:resource`; grants: confirm the id scheme (user/group +
resource). Confirm all three.

### Q1.11 — Size budget
~800 MB today for ~115k resources with fragments; two generations exist briefly during a rebuild.
What is acceptable on the target server, and does that rule anything out (Q1.1, Q1.6)?

### Q1.12 — Complete-indexed fields
Core stores a complete-indexed value as one keyword, never split, so only a single-token value is
reachable by word (K9). Represent it as a non-tokenised string (one token per value) so `field:value`
phrase forms and wildcards still reach multi-word values, or treat the field as ordinary text and
accept the difference? No such field exists on the test system; the fixture needs one (SK3).

### Q1.13 — The 500-character cap
Core turns only the first `$node_keyword_index_chars` characters of a value into keywords, but a
wildcard reads the whole value (K10). Index two representations (the capped text for word matching,
the whole value for wildcards), cap everything, or index everything and accept that a word past the
cap matches in the plugin but not in core (harness A27: 1 / 2)?
**Direction from the acceptance rule (Q10.3, 2026-10-16):** a word past the cap must not match,
so the word-matching representation is capped like core's keywords.

### Q1.14 — The registry as the extension contract
Every index field as an entry in one registry (name or pattern, type, flags, the field types and
index flags it applies to, the value function, source, version, owning package), walked by a generic
builder, with a six-step protocol for a package that needs a field (entry, register row, version
bump, battery cases, builder-path equality, rebuild on the fixture) — see the contract at the top of
[tasks/B-indexing.md](../tasks/B-indexing.md). Confirm, or prefer explicit per-field code in the
builder (simpler to read, every later package then edits the builder).

### Q1.15 — The `0` sentinel in `nodes[]` and `populated_field_ids[]`
The old indexer prepended `0` so the two arrays always exist (`0` is never a real id). Keep it, or
drop it and make the arrays optional — a filter on a missing array matches nothing with
`validate_field_names=0`, which is the wanted answer, and the documents are cleaner. Recommendation:
drop it, with a battery case for a resource that has no nodes at all.

## Done when

A schema document lists every field, its type, source, flags and the reason it exists; the schema
builder and its version number implement it; a rebuild from scratch produces an index whose counts
match MySQL exactly.
