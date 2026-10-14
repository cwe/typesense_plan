# SA — Keywords (catalogue group A)

Free text: words separated by spaces or commas, ANDed, matched over every indexed field the user may
see. Twelve packages. SA1 is milestone M2 (the first served search); the rest are M6, in the order
below unless a question moves one.

How the rows were measured, and what the old plugin returned for each, is in the catalogue
(`docs/core-search-catalogue.md` on `typesense_search_catalogue`); the row IDs here are its IDs. The
per-row plan is in [coverage.md](coverage.md).

### SA1 — Plain words
- **Rows:** A1 A2 A3 A4 A32 A33 H24 · cross: I4 I5 (D2), A5 (SA12), A17/A18 (SA2)
- **Builds:** `search/keywords.php` — the plain-word path: `q` from `$keywords`, lower-cased, every
  word required; `query_by` from the visible indexed fields (D4); the explicit query parameters.
- **Size:** S · **Depends on:** D1–D4, D9, D10, D11 · **Milestone:** M2
- **Core:** one union per word, all must match; a comma is a separator; lower-cased; duplicates
  removed and the string trimmed before the hook; `*` alone and the empty string are the empty
  search (every resource the filters allow).
- **Serve:** `q=<words>`, `drop_tokens_threshold=0`, `prefix=false` for every field unless SA5
  says otherwise, `num_typos` per Q2.2, `split_join_tokens=off`; an empty `q` → `*` with no
  `query_by`, ordered by D9's keyword-less rule.
- **Must honour:** `$keywords` arrives already split, lower-cased and de-duplicated by core; a word
  core cannot find in its keyword table never reaches the plugin (SA11); `prefix` left unset makes
  Typesense prefix-match the last word (`land` 3,172 / 69,408); `split_join_tokens` retries
  zero-hit searches by joining or splitting words (`sun set` → `sunset`); accents: MySQL's
  collation folds them, Typesense folds them too, but the local harness cannot check this
  (SQLite) — one live case.
- **Fixture:** the base fixture.
- **Battery:** `harness/ab/SA1_plain_words.php` — cases A1–A4, A32, A33, I4, I5, H24, plus the
  harness's own: `SUNSET`, `red car` (order-insensitive), two words where one is a prefix of
  another word in the index (`U1`: 2 / 29 with the default prefix), `sun set`; live: the rows.
- **Blocking questions:** Q2.1, Q2.2, Q2.3 (the prefix default only)
- **Open questions:** Q2.18 (accent folding is accepted as parity without a local test?)

### SA2 — Stop words
- **Rows:** A17 A18 · cross: B18 (SB5), A8 (SA3)
- **Builds:** stop-word removal from `q` using core's `$noadd` list.
- **Size:** S · **Depends on:** SA1 · **Milestone:** M6
- **Core:** a `$noadd` word is skipped; `the sculpture` is `sculpture`; a search of only stop words
  has no terms left and returns everything.
- **Serve:** strip `$noadd` words before building `q`; only stop words → the empty search.
- **Must honour:** Typesense has no stop-word list of its own in this schema, so the words are
  required unless removed (`the sculpture` 49,468 / 17,228; `the` 100,862 / 32,537); the list is
  `$noadd` in `definitions.php` plus any configured additions; InnoDB's own stop words apply only
  to the full-text branch (SA8), not here.
- **Fixture:** the base fixture (`the sunset`, `sunset of harbour`).
- **Battery:** `harness/ab/SA2_stop_words.php` — A17, A18, harness A3/A4/N29 and the field case
  from SB5 run here too.
- **Blocking questions:** Q2.4

### SA3 — Quoted phrases
- **Rows:** A6 A7 A8 I8 H14 · cross: A10 H23 (SA4), B3 (SB2), K7 (SK1)
- **Builds:** the phrase path: a quoted term becomes a Typesense phrase in `q`, with the stemming
  rule from Q2.5 / SA12.
- **Size:** M · **Depends on:** SA1, SA12 · **Milestone:** M6
- **Core:** a quoted single word is an exact, unstemmed word; a phrase is words at consecutive
  positions within one value of one field; a stop word inside a phrase is skipped and the position
  gap widened by one (`"black and white"` also matches "black or white"); an unbalanced quote is
  not a quoted string — the quote is a separator (I8 = A1).
- **Serve:** `"a b"` in `q` is a Typesense phrase (adjacent tokens, one field); a quoted single
  word must not be prefix-matched or stemmed; the stop-word gap cannot be reproduced exactly
  (Q2.4 decides: phrase of the non-stop words, or veto).
- **Must honour:** with `$stemming` on the index holds stems and phrase tokens are not stemmed,
  so every phrase returned 0 (`"sculpture park"` 6,883 / 0; harness S9–S13); a phrase in core is
  confined to one value of one field — Typesense phrase matching is per field too, but a
  multi-value field (`_ss`, `_q`) is one array: check adjacency does not cross values.
- **Fixture:** the base fixture has `red car` / `car red` and `foo and bar`; add a value whose
  phrase spans two values of one multi-value field.
- **Battery:** `harness/ab/SA3_phrases.php` — A6, A7, A8, I8, H14 and the stemming cases S9–S13
  with `$stemming` on and off.
- **Blocking questions:** Q2.5, Q2.4
- **Open questions:** Q2.14 (pre-stemming on both sides would make this package trivial)

### SA4 — Negation
- **Rows:** A9 A10 H13 H15 H23 · cross: B16/B17 (SB5), K6 (SK2); H23 needs SA3 first
- **Builds:** `-word` and `-"phrase"` in `q`.
- **Size:** S · **Depends on:** SA1, SA3 · **Milestone:** M6
- **Core:** `-word` is everything except resources holding the keyword (a NOT IN subquery over
  **every** field including hidden and inactive ones); `-"phrase"` likewise over phrases.
- **Serve:** Typesense `-word` (verified); `-"a b"` per Q2.6 (verify Typesense accepts a negated
  phrase before deciding; today it is dropped and the search returns everything).
- **Must honour:** core's NOT reads keyword rows of inactive fields that still exist (`-landscape`
  32,045 / 32,078: the 33 come from an inactive field) and of fields hidden from the user
  (harness A18: 0 / 1) — Q6.7 decides whether the index keeps those words for negation only;
  a negation-only search (`-sunset`) is a match-all with an exclusion, ordered by D9's
  keyword-less rule.
- **Fixture:** the base fixture (hidden field 93, inactive field 94).
- **Battery:** `harness/ab/SA4_negation.php` — A9, A10, H13, H15, H23, harness A16–A19, A18
  (hidden), the inactive-field negative.
- **Blocking questions:** Q2.6, Q6.7

### SA5 — Wildcards
- **Rows:** A11 A12 A13 A14 H22 · cross: B6/B7 (SB3), C5 (SC3), K2/K3 (SK1)
- **Builds:** `word*` as a prefix on that word; the rules for a leading, middle, short or second
  wildcard; the candidate caps.
- **Size:** M · **Depends on:** SA1 · **Milestone:** M6
- **Core:** `sculpt*` is a MySQL full-text `+sculpt*` over node names of **every** visible field,
  indexed or not (51,007); `*scape` is an RLIKE at a word boundary (68,601); `sc*pture` is sent
  as `+sc*pture` and MySQL honours only the trailing part, so it behaves as `sc*` (69,679); a
  wildcard on a word under the full-text minimum length (`so*`) is a LIKE on the keyword table
  instead, so it reads indexed fields only (15,618); two wildcards are two unions (H22 36,026).
- **Serve:** `word*` → `prefix=true` for that word (one boolean for the search — a per-field list
  with more than two fields returned HTTP 400 — and the wildcarded word moved last, P30), with
  `max_candidates` raised from 10 to the value Q2.3 sets (`zeb*` 41 / 10); two wildcards: decide
  rather than veto (H22); leading and middle wildcards per Q2.3 / Q2.17; the short-wildcard
  branch per Q2.16.
- **Must honour:** full parity for A11 needs every field's values in the index (Q1.1) because
  core reads non-indexed fields here (K2 `nikon*` 23,412 / 0); leading wildcards could be served
  from fragments on partial-indexed fields only (P35) — nowhere else without an infix index;
  `max_filter_by_candidates` (4) caps `field:word*` (SB3).
- **Fixture:** the base fixture plus the 40-completion word set from harness 23 (Q1, Q5, Q6).
- **Battery:** `harness/ab/SA5_wildcards.php` — A11–A14, H22, harness A21–A25, Q1, Q4–Q6, U5,
  `$wildcard_always_applied` on and off.
- **Blocking questions:** Q2.3, Q1.1
- **Open questions:** Q2.16 (short wildcards), Q2.17 (wildcard inside a word)

### SA6 — Punctuation, hyphens, apostrophes and commas
- **Rows:** A19 A20 I2 · cross: B10/B11/I3 (SB4)
- **Builds:** the treatment of a single token that core would quote.
- **Size:** S · **Depends on:** SA1, SA3 · **Milestone:** M6
- **Core:** separators (`$config_separators`: hyphen, apostrophe, dots …) split a word into
  words that are ANDed (`black-and-white` = black AND and AND white, with the stop word dropped;
  `atkinson's` = atkinson AND s); a string with **no whitespace** is kept as one keyword and its
  comma counts as punctuation, so `sculpture,landscape` is the **phrase** "sculpture landscape"
  (22), not two words (34,408); a punctuated single token next to other terms is split instead.
- **Serve:** the plugin receives core's split `$keywords`, so most of this is parity by
  construction once `token_separators` matches `$config_separators` (P31); the no-whitespace comma
  case and `foo-bar`-as-a-phrase need the rule in Q2.8 (quote it as a phrase, as core does, or
  simplify to AND).
- **Must honour:** `token_separators` must be single-byte (Typesense rejects curly quotes);
  space is never a separator; an email address and a filename are single tokens in the harness
  (A10, A12 pass today).
- **Fixture:** the base fixture (`foo-bar`, `O'Brien`, `john.smith@example.com`, `IMG_1234.jpg`).
- **Battery:** `harness/ab/SA6_punctuation.php` — A19, A20, I2, harness A8–A12, N7a, N7b, O16.
- **Blocking questions:** Q2.8

### SA7 — OR groups
- **Rows:** A15 A16 H32 H33 B5 H34 C25 · cross: C6 (SC2: an OR of option names is resolved by core
  into one node bucket, not an OR of words)
- **Builds:** recognition of `;` in a word or a field value and the decision for each form.
- **Size:** S · **Depends on:** SA1, SB1 · **Milestone:** M6
- **Core:** `title:a;b` is either keyword in that field (an IN over alternative keywords); a bare
  `a;b` is looked up as the single keyword "a;b", which exists only after a field OR search has
  created it as a side effect (Core 9) — first run a suggestion, later runs an OR; `land*;sun*`
  forces the RLIKE branch `(land.*?|sun.*?)`; the documented `country:united kingdom;spain` binds
  only the first word, so the rest becomes a bare OR and a suggestion (C25).
- **Serve:** field-scoped OR (`title:a;b`) as two ORed field filters (`field_8_s:a || field_8_s:b`)
  if Q2.7 says so — Typesense `filter_by` has a real OR, `q` does not (`drop_tokens` is not an OR:
  `sunset;beach` AND-matched to 0); bare `a;b` and wildcard OR: veto (core's own behaviour is a
  bug and a suggestion respectively).
- **Must honour:** never send `;` inside `q` (P4); a veto for the bare form hands the search to
  core, which then shows its suggestion — the user sees the same thing either way.
- **Fixture:** the base fixture (`ship;book`).
- **Battery:** `harness/ab/SA7_or_groups.php` — A15, A16, B5, H32–H34, C25, harness A20, B26.
- **Blocking questions:** Q2.7

### SA8 — The full-text box
- **Rows:** A27 A28 A29 A34 J12 K4 K8
- **Builds:** recognition of `"@FULL_TEXT:…"` (`FULLTEXT_SEARCH_PREFIX`) and the veto.
- **Size:** S · **Depends on:** D2 · **Milestone:** M6
- **Core:** the advanced search's full-text box sends `"@FULL_TEXT:words"`: a MySQL boolean-mode
  MATCH over node names of every visible field, indexed or not, with `+`, `-` and `[QUOTES]`
  passed through; words without an operator are **OR**ed (`distant island` 956, `+distant
  +island` 9).
- **Serve:** veto, permanently, unless Q2.15 chooses otherwise: Typesense cannot OR words in `q`,
  and the box's semantics are MySQL's.
- **Must honour:** the veto must fire whether the term is alone or appended to other terms (J12);
  the plugin must recognise the prefix inside the quotes after core's normalisation.
- **Battery:** `harness/ab/SA8_full_text.php` — each row declines with the reason "full-text
  boolean search"; the harness emulates MATCH so core's side is an emulation (mark it so).
- **Blocking questions:** —
- **Open questions:** Q2.15 (confirm the permanent veto and the user-facing note on the advanced
  search page)

### SA9 — Words that are not metadata
- **Rows:** A23 A24 A25 A26 H36 H37 · cross: F30/F31 (SF4)
- **Builds:** type names, contributor usernames, bare numbers and bare years as search words.
- **Size:** M · **Depends on:** SA1, B2 (a registry entry for the words), D9 · **Milestone:** M6
- **Core:** a word equal to a resource type's name matches every resource of that type as well
  as the word (`document` 490); with `$index_contributed_by` a username matches every upload by
  that user (off on the test system: A24 is a plain word there); a bare number with
  `$config_search_for_number` off is the word plus that resource, ordered first (A25 100 / 0);
  with it on, the same as `!resource`; a bare year matches dates in any date field because dates
  are indexed as year, year-month and full-date keywords (A26 10,289 / 0).
- **Serve:** per Q2.9 / Q1.7 — index type names and usernames as words on the resource (one array
  field, in `query_by`), or veto any search containing one; bare year: add the date `_q`
  representations of every date field to `query_by` for a numeric-looking word (Q2.11); bare
  number: `_eval(ref:=N):desc` as the first sort key, verified live (Q7.4), and `ref_s` in
  `query_by` so the word itself matches.
- **Must honour:** core applies the type-name rule to the first word only? — verify in
  `do_search_keywords.php:56` before writing the battery; `$index_contributed_by` indexes the
  username at upload time in core, so parity after a username change differs either way.
- **Fixture:** the base fixture (types Photo/Document/Video/Audio, users admin/tester/other, a
  resource whose ref is also a word in another resource: harness A7).
- **Battery:** `harness/ab/SA9_non_metadata_words.php` — A23–A26, H36, H37, harness A5–A7, I1
  (contributed by), with both settings of `$config_search_for_number` and `$index_contributed_by`.
- **Blocking questions:** Q2.9, Q1.7, Q2.11, Q7.4

### SA10 — Related keywords
- **Rows:** A21 A22
- **Builds:** synonym handling for `keyword_related`, or the veto.
- **Size:** S · **Depends on:** SA1, C7 (the related-keywords event, E12) · **Milestone:** M6
- **Core:** a keyword's related keywords are ORed into its union, in both directions on the test
  system (antony ↔ anthony, 1,491 each way).
- **Serve:** a Typesense synonym set (30.x has synonym sets, not per-collection synonyms) kept in
  step by the hook, multi-way or one-way to match core's direction; or veto any search containing
  a keyword that has related entries (Q2.10).
- **Must honour:** the old synonym sync never worked; synonyms interact with stemming and with
  phrases (a synonym inside a phrase?) — test both.
- **Fixture:** the base fixture (`automobile` → car, harness A26/A44).
- **Battery:** `harness/ab/SA10_related_keywords.php` — A21, A22, harness A26, A44, a related
  word inside a phrase, after a related-keyword edit (freshness).
- **Blocking questions:** Q2.10

### SA11 — Unknown words and suggestions
- **Rows:** A30 A31
- **Builds:** nothing in the plugin; a battery that records the dependency.
- **Size:** S · **Depends on:** D1 · **Milestone:** M2
- **Core:** a word absent from the keyword table ends the search before the hook with a
  sound-alike suggestion (a string, for integer `fetchrows`) or an empty array; the plugin is never
  consulted.
- **Serve:** nothing to serve; the plugin's results depend on core's keyword table being complete
  for the words a user types (Q2.12). The battery documents it so a later change to core (Q11.5)
  is noticed.
- **Battery:** `harness/ab/SA11_unknown_words.php` — A30, A31, harness 12 (row shape of the
  suggestion), "not consulted" asserted.
- **Blocking questions:** Q2.12

### SA12 — Stemming and the stemmer
- **Rows:** A5 · cross: every text row under `$stemming`
- **Builds:** the stemming rule in the schema (A3) and in `q`: Typesense's stemmer, or RS's own
  stemmer applied on both sides.
- **Size:** M · **Depends on:** A3, A5 (text), B2 (the stem flag) · **Milestone:** M2 (the rule must exist before
  the first index)
- **Core:** with `$stemming` on, every indexed word is stored stemmed by `GetStem()`
  (`lib/stemming/<language>.php`: an irregular-plural table, then Porter) and the original is kept
  when the stem differs; query words are stemmed the same way; quoted words are not stemmed.
- **Serve:** today text fields are declared `stem: true` (Snowball) iff `$stemming` (P31) — 34 of
  40 word-form pairs agree with RS's stemmer, the irregular plurals differ. The alternative is to
  stem with RS's `GetStem()` at index time and at query time and leave Typesense's stemmer off:
  identical stems by construction, and phrases (SA3) and quoted words need no special rule.
- **Must honour:** `_q`, `_ss` and sort keys are never stemmed; the stemmer must be the system
  default language's, not the user's (core's rule); a `$stemming` change is a full rebuild (B9, E25).
- **Fixture:** the base fixture under `HARNESS_COMBO_STEMMING` and the 13_stemming word-form list.
- **Battery:** `harness/ab/SA12_stemming.php` — A5, the 40 word-form pairs, S1–S14, the
  combinations battery with stemming on.
- **Blocking questions:** Q1.5
- **Open questions:** Q2.14 (pre-stem with RS's stemmer on both sides?)
