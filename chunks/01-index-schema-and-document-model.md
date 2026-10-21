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

**Catalogue marks (2026-10-20):** K2 (`nikon*`) and K3 (`camera:nik*`), the wildcard forms over a field not
flagged for indexing, are *check*; K1 and K7 are *must* and are answered by core before the hook.

**Measured (2026-10-20), active fields, raw text held in node names:** on the test system (114,596
resources) the flagged text fields hold 10.3 MB (23 fields, 472,221 values), the flagged option names
25.1 MB, the flagged dates 2.0 MB; the non-flagged text fields hold 26.7 MB (5 fields, 209,935 values),
of which `accessibilityalttext` (field 86, multi-line, a value on 114,100 resources, about 225 characters
each) is 25.7 MB and `camera` 0.9 MB (451 distinct values on 76,743 resources). On the second, larger
test database (1,377,433 resources) the flagged text fields hold 493.7 MB (78 fields, 20.2 M values,
values up to 4,477 characters), the flagged option names 122.6 MB; the only non-flagged text field is
`camera`, 16.5 MB (490 distinct values on 1,143,498 resources). So option (b) adds 2.6× the flagged text
on the first system (one field) and 3 % on the second.

**Decided 2026-10-20: option (b), for now.** Every active field's values are indexed; the index flags decide
which fields a plain keyword search, a field term or an option word reaches, while wildcards and the
full-text box reach every visible field, as core does. Revisited when tested against different data
sets (H8). An exclusion list set on the setup page (G9) keeps named fields out of the index entirely,
and the searches that could reach an excluded field decline for users who can see it.

### Q1.2 — Which representations per field type?
Keep the current set? Specific points: `_q` for text fields is unused (the bare-colon filter on
`_s`/`_text` does the job) — drop it; add a sortable key per field that core can sort by (title,
custom `field<n>` with numeric and dot-notation rules, status) — which fields; keep `_ts` when
`_range_start` covers every case.

**Decided 2026-10-20: option (a).** Version 1 is the old set minus its dead weight — `_q` on text fields,
`_ts` and the facet flag on `_s` go — and nothing speculative. A representation enters the schema only
with the package that proves it; additions are batched into one rebuild per release. `_k` waits for
Q3.4, the sort keys for Q7.2, the date components for Q3.13, `_c` and `_w` for Q1.12 and Q1.13.

### Q1.3 — Fragments: keep, and should they be free-text searchable?
Fragments are in `query_by` for partial-indexed fields, so a word held only in a fragment is
findable. Catalogue E14 says fragments are *not* free-text searchable — reconcile (one of the two is
out of date) and confirm the intended behaviour. Also confirm the memory cost is acceptable on the
target server.

**Catalogue marks (2026-10-20):** K5 (a bare fragment of a partially indexed field) is *must* — SB6 serves it.

**Decided 2026-10-21: option (a).** Fragments stay and are in `query_by` for every partial-indexed field;
a bare fragment matches as in core (K5). Reconciled: core treats fragments as ordinary keywords; the
"not free-text searchable" note described the old plugin. The memory cost is Q1.11's, not a limit.

### Q1.4 — Value cleaning at index time
(a) Strip HTML tags and entities before indexing (core strips? verify) — yes/no; (b) translated
values: index every language, only the default, or per-language fields; (c) cap at
`$node_keyword_index_chars` like core, or index whole values (more matches than core); (d) keep the
invisible-character stripping. Note the test data has none of (a)–(c), so decide on principle.

**Decided 2026-10-21: option (a) — do what core does at each site.** Tags stripped as core strips them
(`<a>` kept), every translation indexed, the 500-character cap on keyword representations (`_w` only
if K10 requires it), invisible characters stripped as `cleanse_string` does. Decided on principle:
neither test database holds the data.

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

**Decided 2026-10-20: option (b).** Version 1 is the base rows plus the hit-count cluster (`hit_count`,
`user_rating`, `rating`, the four packed keys), the cluster conditional on Q7.1 accepting rebuild-fresh
ordering and on the G11 check; everything else enters with its package, batched per release (Q1.2).

### Q1.7 — Resource-type names and contributor usernames as searchable words
Core matches "Photo" to every resource of that type, and usernames when `$index_contributed_by` is
on. Index a `type_name` (and `contributor`) token field, or veto searches containing a type name?

### Q1.8 — Schema evolution
Confirm generation collections + aliases as the mechanism (prior decision P7); what triggers a full
rebuild (field config change, `$stemming` change, plugin schema version change); how the schema
version is recorded (alias name suffix, a metadata document); whether `PATCH` upgrades remain for
additive changes between rebuilds.

**Decided 2026-10-20: rebuild only.** A change to the collection definition is a generation rebuild; the
live collection is never altered in place (no PATCH). Version in the generation metadata; a mismatch
is "rebuild required", served from the old generation meanwhile. Decides schema changes only: which
document-level changes are streamed, queued or left to the rebuild belongs to the sync questions
(Q8.x), decided later; B9 carries a candidate change map for them.

### Q1.9 — Typesense version and features
Pin to 30.x and use synonym sets, `_eval` sorts, `multi_search`, reference joins, `cascade_delete`,
aliases. Any reason to support an older server?

**Decided 2026-10-20: option (a).** Minimum Typesense 30.0, tested on 30.x; refuse to activate below the
minimum, warn on an untested later major; the version check at setup, shown on the setup page; the
features above used without compatibility layers.

### Q1.10 — Document identity
Resources: `id = ref`; memberships: `collection:resource`; grants: confirm the id scheme (user/group +
resource). Confirm all three.

**Decided 2026-10-20: natural keys, confirmed for all three.** Resources `id` = the ref as a string (plus
an integer `ref` and `ref_s`); memberships `collection:resource`; grants `resource:user:usergroup`
with `-1` in the unused slot. One id function per collection in B2, shared by import, queue and delete.

### Q1.11 — Size budget
~800 MB today for ~115k resources with fragments; two generations exist briefly during a rebuild.
What is acceptable on the target server, and does that rule anything out (Q1.1, Q1.6)?

**Measured (2026-10-20):** the second test database holds 494 MB of flagged text and 123 MB of option
names across 1.38 M resources against the test system's 10 MB and 25 MB across 115 k — roughly 40× the
text for 12× the documents, with values up to 4,477 characters (the 500-character cap, K10, matters
there). If the test index's 800 MB resident scales with the text, that installation needs several GB;
the budget question is this one, not Q1.1's extra field.

**Decided 2026-10-20: no budget rule.** The server must have room to serve one generation and build the
next beside it; B8 checks the memory before a build and refuses when a second generation would not fit;
the setup page shows the generation's size; G8 gives sizing guidance from the measured reference
points. The levers stay documented, nothing is cut in advance.

### Q1.12 — Complete-indexed fields
Core stores a complete-indexed value as one keyword, never split, so only a single-token value is
reachable by word (K9). Represent it as a non-tokenised string (one token per value) so `field:value`
phrase forms and wildcards still reach multi-word values, or treat the field as ordinary text and
accept the difference? No such field exists on the test system; the fixture needs one (SK3).

**Catalogue marks (2026-10-20):** K9 (complete indexing) is *check*; no such field exists on the test system,
so the check needs the fixture.

### Q1.13 — The 500-character cap
Core turns only the first `$node_keyword_index_chars` characters of a value into keywords, but a
wildcard reads the whole value (K10). Index two representations (the capped text for word matching,
the whole value for wildcards), cap everything, or index everything and accept that a word past the
cap matches in the plugin but not in core (harness A27: 1 / 2)?
**Direction from the acceptance rule (Q10.3, 2026-10-16):** a word past the cap must not match,
so the word-matching representation is capped like core's keywords.

**Catalogue marks (2026-10-20):** K10 (a word beyond the 500th character) is *check*.

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
