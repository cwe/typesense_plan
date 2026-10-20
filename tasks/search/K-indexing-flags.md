# SK — What the indexing flags control (catalogue group K)

Only the values of fields flagged for indexing become keywords in core, so the flag governs every
form that goes through the keyword table: plain words, field words, phrases, OR, NOT and the
short-wildcard branch. It does **not** govern full-text and other wildcards (node names), nor date
and number terms, options by name, `@@` tokens, `!hasdata` and `!empty` (nodes). Three packages,
milestone M4; SK1 is the slice of Q1.1 that the catalogue makes concrete.

**Catalogue marks (2026-10-20):** K2, K3, K4, K8, K9, K10, K11 and K12 are *check*; K1, K5, K6 and K7 are
*must* (K1 and K7 are answered by core before the hook, K6 is P37, K5 is SB6).

### SK1 — Fields not flagged for indexing
- **Rows:** B12 B13 B19 K1 K2 K3 K7 · cross: C9 (SC3), D22 D23 (SD5), K11 (SE1), K4 K8 (SA8)
- **Builds:** B2's selector for which representations a non-indexed field gets, and D4/SB1's
  `query_by` and field-filter behaviour for it.
- **Size:** S · **Depends on:** B1, B2, SB1, SA5 · **Milestone:** M4 (a rebuild)
- **Core:** a word that exists only in a non-indexed field is not a keyword: nothing, and a
  suggestion (K1, K7 — core returns before the hook); `field:word` on such a field is nothing
  although the values are there (B12, B13, B19); a word of an option of a non-indexed fixed-list
  field is nothing (C9); but `nikon*`, `camera:nik*` and the full-text box read node names and
  match (K2, K3, K4: 23,412); dates and numbers join nodes and work (D22, D23, K11).
- **Serve:** two consistent choices — (a) index values of flagged fields only: plain words,
  field words and options are parity by construction; wildcards over non-indexed fields (K2, K3)
  cannot be served → veto a bare `word*` when any visible non-indexed text field exists, or accept
  the difference; dates and numbers indexed regardless (SD5). (b) index every field's values
  but keep non-indexed fields out of `query_by` and out of the word filters, using them only for
  wildcards: full parity, more memory (Q1.11).
- **Must honour:** the old plugin had no attributes for non-indexed fields at all (D22 3,676 / 0;
  O1–O3); a `field:value` filter on a field the index does not have must match **nothing**
  (`validate_field_names=0`), never veto to a slower core answer of nothing — unless Q3.6 prefers
  the veto; the advanced search lists indexed fields only, the simple bar needs indexed or
  fixed-list.
- **Fixture:** the base fixture's field 96 (non-indexed multi-line); add a non-indexed
  single-line text field, a non-indexed dropdown (as `source` is), a non-indexed date and numeric
  field.
- **Battery:** `harness/ab/SK1_non_indexed_fields.php` — the rows above, harness A24, O1–O3,
  21_non_indexed cases.
- **Blocking questions:** Q1.1, Q3.6, Q2.3

### SK2 — Inactive fields and fields hidden by permission
- **Rows:** K6 K12 · cross: A9's 33 (SA4), Q6.7
- **Builds:** B1 marks inactive fields and B4 excludes them from the attributes; D4 excludes hidden and inactive fields
  from `query_by`; SB1 returns false for a hidden field; the negation rule for their words.
- **Size:** S · **Depends on:** B1, D4, SB1, SA4 · **Milestone:** M4
- **Core:** an inactive field is found by name but counts as hidden and is excluded from the
  union: `description:sculpture` is nothing (K6); a field hidden by permission makes a field term
  return false before the plugin is asked, and its values are excluded from every union (K12);
  **but** NOT reads the keyword rows of both (A9 32,045 / 32,078; harness A18, "negative on a word
  only in an inactive field").
- **Serve:** P37 (inactive fields out of positive matching and out of the attributes) plus the
  hidden-field exclusion in `query_by` (gated on `metadata_field_view_access`, which needs
  `RS_TEST_MODE` on the CLI); negation over those words per Q6.7 — keep the words in a
  negation-only field, or accept the difference.
- **Fixture:** the base fixture's fields 93 (hidden by `f-93`) and 94 (inactive, indexed while
  active).
- **Battery:** `harness/ab/SK2_inactive_hidden_fields.php` — K6, K12, harness A18, A35, A36, B17,
  14_inactive cases.
- **Blocking questions:** Q6.7 (confirm P37)

### SK3 — Complete indexing and the 500-character cap
- **Rows:** K9 K10
- **Builds:** A5's `cap()` and the complete-indexing representation (`_c`) in B2.
- **Size:** S · **Depends on:** A5, B2 · **Milestone:** M4
- **Core:** complete indexing stores the whole value as one keyword, never split, so a value
  with spaces cannot be typed as a word — only a single-token value is reachable by word, and
  wildcards or full text otherwise (K9; no such field on the test system); only the first
  `$node_keyword_index_chars` (500) characters of a value become keywords, a wildcard still reads
  the whole value (K10; harness A27/A28: `zeppelin` past the cap 1 / 2 — the old index held the
  whole value).
- **Serve:** complete-indexed fields: one token per value (a non-tokenised string field, or the
  value with spaces replaced by a non-separator) so a single-token value matches by word and
  multi-word values match only through `field:value` phrase forms (SB2) and wildcards; the cap:
  index the first 500 characters for word matching and the whole value for wildcards (two
  representations), or accept the difference.
- **Fixture:** a complete-indexed field with a one-word and a two-word value; the existing
  600-character caption (resource 6).
- **Battery:** `harness/ab/SK3_complete_index_cap.php` — K9, K10, harness A27, A28, A48.
- **Blocking questions:** Q1.4
- **Open questions:** Q1.12 (complete indexing representation), Q1.13 (two representations for
  the cap, or accept)
