# SB — Field-specific terms on text fields (catalogue group B)

`shortname:value` binds one word to one field; the rest of a multi-word value is free text unless
the whole term is quoted. Six packages, milestone M4. SB1 is the hand-off point from D4's field
token split; everything in SC, SD and SE goes through the same lookup.

### SB1 — Field lookup and `field:word`
- **Rows:** B1 B8 B15 B20 B21 H3 H25 I6 I7 I11 · cross: B9 (SC3: the value is a dynamic-keyword word), B19 B12 B13 (SK1)
- **Builds:** the field lookup in `search/criteria.php` — short name → ref, type, `keywords_index`, `partial_index`,
  `active`, `field_constraint`, viewability (cached, and the lookup *returns* every flag); in
  the same file the text-field filter.
- **Size:** M · **Depends on:** D4, B1 · **Milestone:** M4
- **Core:** the name is matched whole and case-insensitively (`Title:Sculpture` = B1; a name may
  contain a slash: `photographer/videographer`); an unknown name turns the colon into a space, so
  `foo:sculpture` is the two words foo and sculpture (B15 — and a word core does not know gives a
  suggestion, SA11); a trailing or leading colon is the same (I6, I7); the value is one word,
  matched through the keyword union restricted to that field's nodes; two terms on one field or
  on two fields are ANDed (B20, B21).
- **Serve:** `field_<ref>_text:<word>` (multi-line) or `field_<ref>_s:<word>` (single-line) — the
  bare-colon operator is a tokenised word-contains filter that honours the field's tokeniser and
  stemming, so it returns what a keyword search of that field returns (`title:Gallery` 1,982 =
  `q=Gallery&query_by=title`); the token is removed from `q`; values with spaces or punctuation
  are backtick-quoted; unknown names fall through to plain words.
- **Must honour:** a field the user cannot view returns false in core before the hook (K12), and
  its values must not be in `query_by` either; an inactive field is hidden (SK2, P37); the field
  with short name `date` exists on the test system — the lookup, not the name, decides what a
  token is; a numeric field's plain value goes to `_q` (SE1); partial-indexed fields filter on
  `_p` (SB6); non-indexed fields per SK1.
- **Fixture:** the base fixture (title, caption, credit? — add `credit` field 10 as in the
  combinations fixture; a field named with a slash).
- **Battery:** `harness/ab/SB1_field_word.php` — B1, B8, B9, B15, B20, B21, H3, H25, I6, I7, I11
  and a lookup table test over every flag combination.
- **Blocking questions:** Q3.11
- **Open questions:** Q3.17 (the exact case and character rules of the name match)

### SB2 — Multi-word values and quoted field terms
- **Rows:** B2 B3 B4 · cross: C2/C8/C11 (SC2), the search bar's rebuilt text (SJ1)
- **Builds:** `"field:two words"` (the advanced search and the search bar produce it) and
  `field:"x y"`.
- **Size:** S · **Depends on:** SB1, SA3 · **Milestone:** M4
- **Core:** `title:henry moore` binds henry to title and leaves moore as free text (251);
  `"title:henry moore"` is a phrase search restricted to the field (251 — the form the advanced
  search sends); `title:"henry moore"` is not a supported form but the stray quotes are stripped
  and it works out as B2.
- **Serve:** B2 by construction (core splits the tokens); B3 as a phrase filter on the field —
  the bare-colon operator takes a backtick-quoted multi-word value, but whether it is a phrase
  (adjacent) or an AND of words must be verified live before choosing (if it is an AND, the row
  is a ≈ difference or a veto); B4 as B2.
- **Must honour:** the quoted token stays in `$search` and must be recognised before SA3 treats
  it as a free-text phrase (today → `q` → 0); for fixed-list fields the same token was resolved by
  core into the node bucket and only needs dropping (SC2).
- **Fixture:** the base fixture (`"title:launch party"`).
- **Battery:** `harness/ab/SB2_quoted_field_terms.php` — B2, B3, B4, harness B2, B3, S2, S3, S7.
- **Blocking questions:** Q3.2
- **Open questions:** Q3.18 (is the backtick-quoted bare-colon value a phrase or an AND?)

### SB3 — Wildcards in a field
- **Rows:** B6 B7 H5 · cross: K3 (SK1), C5 (SC3)
- **Builds:** `field:word*` and `field:*word`.
- **Size:** S · **Depends on:** SB1, SA5 · **Milestone:** M4
- **Core:** `title:sculpt*` is a full-text MATCH restricted to the field, reading node names
  (so it works on a non-indexed field too: K3); `title:*scape` is an RLIKE restricted to the field.
- **Serve:** `field_<ref>_text:sculpt*` (prefix on the word, verified) with
  `max_filter_by_candidates` raised from 4 (`title:zeb*` 40 / 4); the leading form per Q2.3.
- **Must honour:** the candidate cap silently truncates — raise it or veto above it (Q3.10).
- **Fixture:** the 40-completion set (harness Q2).
- **Battery:** `harness/ab/SB3_field_wildcards.php` — B6, B7, H5, harness Q2, I3, P17 (a wildcard
  on a date field returns nothing in core).
- **Blocking questions:** Q3.10, Q2.3

### SB4 — Punctuation inside a field value
- **Rows:** B10 B11 I3
- **Builds:** the quoted-term rule for punctuated field values.
- **Size:** S · **Depends on:** SB1, SA6 · **Milestone:** M4
- **Core:** a field value with punctuation is quoted by core and searched as the phrase of its
  parts in that field: `exhibitiontitle:fabric-ation` → "fabric ation" (1,453);
  `dimensions:16.5x16.5cm` → the phrase 16, 5x16, 5cm (20); `title:sculpture,landscape` → the
  phrase "sculpture landscape" in title (0; the old plugin gave 25 by ANDing the words).
- **Serve:** a backtick-quoted phrase on the field (SB2's mechanism), split with the same
  separators core uses.
- **Battery:** `harness/ab/SB4_field_punctuation.php` — B10, B11, I3, harness B6, B9.
- **Blocking questions:** Q2.8, Q3.18

### SB5 — Negation and stop words on a field
- **Rows:** B16 B17 B18
- **Builds:** the three degenerate forms.
- **Size:** S · **Depends on:** SB1, SA2, SA4 · **Milestone:** M4
- **Core:** `-title:sculpture` is not supported: read as `-title` plus `sculpture` (49,241);
  `title:-sculpture` is quoted and the word `-sculpture` omitted, so it excludes nothing (100,862);
  `title:the` is a stop word, skipped: everything (100,862).
- **Serve:** replicate each (they are cheap once SB1, SA2 and SA4 exist) or veto; the old plugin
  vetoed B16 and answered B17/B18 wrongly (0 and 4,226 / 10,564). Going beyond core (a real
  negated field term) is Q3.8.
- **Battery:** `harness/ab/SB5_field_negation_stop.php` — B16, B17, B18, harness N5a.
- **Blocking questions:** Q3.8, Q2.4

### SB6 — Partial indexing
- **Rows:** B14 K5
- **Builds:** `_p` fragments in the field filter and in `query_by`.
- **Size:** S · **Depends on:** A5 (fragments), B2 (the `_p` entry), SB1 · **Milestone:** M4
- **Core:** a partial-indexed field stores every fragment of three or more letters of each word
  as a keyword at the word's position, so `originalfilename:ksho` finds "workshop" (665) and the
  bare fragment `ksho` finds the same 665 through the keyword table (K5).
- **Serve:** `field_<ref>_p:ksho` for the field form (P35); `_p` in `query_by` for the bare form,
  which is Q1.3.
- **Must honour:** fragments are lower-cased substrings ≥ `$partial_index_min_word_length`; mean
  49 per filename, ~108 MB for 114,596 filenames; the plain-word path must not prefix-match on
  `_p` (a fragment is an exact token).
- **Fixture:** the base fixture (field 51).
- **Battery:** `harness/ab/SB6_partial_index.php` — B14, K5, harness A38, A39.
- **Blocking questions:** Q1.3
