# Chunk 2 — Free-text keyword semantics

Status: planning. Depends on: 0, 1. Feeds: 10 (test cases).

## Scope

What a plain keyword search means: AND/OR/NOT, phrases, stop words, prefix and wildcard matching,
typo tolerance, punctuation and commas, synonyms/related keywords, words that are not metadata (type
names, usernames, years), and what to do with a word core cannot find.

## What we know

- Core: every keyword is a node/keyword union; keywords AND; `;` ORs within a keyword; `-word`
  omits over **all** fields including hidden and inactive ones; `"phrase"` checks positions; stop
  words (`$noadd`) are skipped, and a phrase with a stop word only checks distance (`"black and
  white"` matches "black or white"); a search of only a stop word returns everything; `word*` is
  MySQL full text over every node, indexed or not; a single token with punctuation (`foo-bar`) or
  no-whitespace comma string is a phrase; core returns a suggestion (never calling the plugin)
  when a word is not in the keyword table; no typo tolerance; related keywords are ORed in; a
  type name matches every resource of that type; a bare year matches date values as words.
- Plugin: `q` with `drop_tokens_threshold=0` (AND); Typesense honours `-word`; quoted phrases pass
  through; `prefix` was unset so the **last word of every search is prefix-matched** (`land`
  3,172 / 69,408); `max_candidates` caps wildcard completions (`zeb*` 41 / 10); `split_join_tokens`
  retries zero-hit searches (`sun set` → "sunset"); stop words are required (`the sculpture`
  49,468 / 17,228); with `$stemming` on, **quoted phrases return nothing** (`"sculpture park"`
  6,883 / 0 — the index holds stems, phrase tokens are not stemmed); `-"phrase"` is dropped
  (returns everything); OR-groups veto; two wildcards veto; leading/middle wildcards are sent
  literally (`*scape` 68,601 / 381); `foo-bar` is AND of parts; a year as a word finds nothing;
  type names only text-match; the synonym sync never worked (and v30 has no per-collection
  synonyms); `num_typos` is set by the plan (value to check); `-landscape` differs by 10 because an
  inactive field keeps keyword rows in core.
- Stemmer drift 34/40; accents folded by Typesense.

## Candidate work items

- Send explicit query parameters on every search: `prefix` per field (false unless a wildcard),
  `num_typos`, `split_join_tokens=off`, raised candidate caps for wildcards, `drop_tokens_threshold=0`.
- Strip `$noadd` stop words from `q` as core does; decide the phrase-with-stop-word case.
- A policy for phrases under stemming (Q2.5).
- Related keywords via synonym sets, or veto.

## Questions to answer before coding

### Q2.1 — Target: identical sets to core, or sensible semantics?
Core's keyword behaviour includes quirks (stop-word skipping, NOT over hidden fields, the comma
rule, the bare-OR side effect). Identical means copying them. Which quirks are copied, which are
declined (veto), which are improved on (and documented as differences)? This is Q0.3 applied to
keywords; answer per item in the list above.

### Q2.2 — Typo tolerance
Core has none. Set `num_typos=0` for parity, or allow typos as a deliberate improvement?

### Q2.3 — Prefix and wildcards
Set `prefix=false` by default (fixes `land`). For `word*`: raise `max_candidates` (to what — 100,
exhaustive?) and `max_filter_by_candidates` for `field:word*`. Leading/middle wildcards: veto, or
serve via fragments/infix where the field has them? Wildcards over non-indexed fields: core finds
them, the plugin cannot unless Q1.1 indexes everything — veto bare wildcards, index everything, or
accept.

**Catalogue marks (2026-10-20):** A12 (leading wildcard) and B7 (leading wildcard in a field) are *check* —
Chris checks core's behaviour before the marks are final; SA5 and SB3 keep the rows meanwhile.

**Catalogue marks (2026-10-20):** H22 (two wildcards) and K2 (a wildcard over a non-indexed field) are *check*.

**Q1.1 decided (b) on 2026-10-20:** the values of every active field are in the index, so wildcards over
non-indexed fields are served once the mechanism here is settled; the question that remains is the
mechanism for leading and inner wildcards and the candidate caps.

### Q2.4 — Stop words
Strip `$noadd` words from `q` before sending (core skips them). A search that is only stop words:
return everything (core) or nothing? Phrase with a stop word: core's distance-only check cannot be
reproduced exactly — treat the phrase as its non-stop words, or veto phrases containing stop words?

### Q2.5 — Phrases when stemming is on
Options: (a) stem the phrase tokens client-side with the same stemmer as the index (drift risk);
(b) index an unstemmed copy of text fields and search phrases against it (memory); (c) veto quoted
phrases when `$stemming` is on; (d) turn stemming off in the index and stem at query time only
(changes keyword matching). This is the largest single divergence on the test system.

**Catalogue marks (2026-10-20):** H14 (a phrase and a node) is *must* — SA3's phrase must compose with a node
bucket.

### Q2.6 — Negated phrase `-"…"`
Implement (does Typesense accept `-"a b"`? verify) or veto. Today it returns everything, the worst
outcome.

**Catalogue marks (2026-10-20):** A10 (`-"sculpture park"`) is *check* — core's exclusion of the phrase is
confirmed before the mark is final; SA4 keeps the row.

**Catalogue marks (2026-10-20):** H23 (a phrase and a NOT phrase) is *no* — never served; A10 (the NOT phrase
alone) stays *check*.

### Q2.7 — OR-groups `a;b`
Keep the veto. Core's bare `a;b` only works after a field search has created the combined keyword
(Core 9) — a core bug. Field-scoped OR (`title:a;b`) could be served as two ORed filters: do it?
**Direction 2026-10-16 (for now):** field-scoped OR as a filter `||` now; an all-fields OR as the
same filter-OR over the user's visible fields if its operation count fits the raised server limit,
otherwise merged per-alternative searches; relevance order of OR searches an accepted difference.
Still to settle: whether a bare `a;b` should work at all (A16 is marked *check*), and the measured
operation count.

**Catalogue marks (2026-10-20):** A15 (OR of two wildcards) moves from *must* to *check*, alongside A16. The
mechanism decided here stands; whether A15 and A16 are served with it is confirmed after Chris checks core.

**Catalogue marks (2026-10-20):** H33 (a wildcard OR and a node) is *no*; H34 (a field OR and a node) is *must*,
so the mechanism decided here is required for the field form at least; H32 (a bare OR and a node) is a bug
row (Core 9) that Chris would check.

### Q2.8 — Punctuation and commas
`foo-bar` is a phrase in core when the search has no plain-split path, otherwise two words;
`a,b` with no whitespace is a phrase. The plugin receives core's split keywords, so the difference is
only in how it treats a punctuated single token. Replicate (quote it) or simplify?

**Catalogue marks (2026-10-20):** I2 (a comma without a space) is a bug row (Core 10) Chris would check; I3 (a
comma inside a field term) is *check*.

### Q2.9 — Type-name and contributor-name words
Index them (Q1.7) or veto searches containing a type name / username when `$index_contributed_by`?

**Catalogue marks (2026-10-20):** H37 (a type name and a keyword) is *must*.

### Q2.10 — Related keywords
Core ORs related keywords into the union. Options: Typesense synonym sets (one-way or multi-way —
which matches core?), maintained by the existing `after_save_related_keywords` hook; or veto any
search containing a keyword that has related entries. The test system has one pair.

### Q2.11 — Bare years and numbers
A year typed as a word matches date values in core. Add date representations (`_q`) to `query_by`
so `2024` matches dates? Bare numbers with `$config_search_for_number` off: core boosts the exact
ref to the top — implement with `_eval(ref:=N)` (verified) or keep vetoing?

**Catalogue marks (2026-10-20):** H36 (a year as a word and a keyword) is *must*.

### Q2.12 — Unknown words and suggestions
Core returns a suggestion string (integer `fetchrows` only) and never consults the plugin when a word
is absent from the keyword table. Accept that dependency (the plugin's results depend on core's
keyword table being complete) or route such searches to the plugin anyway via a core change?

### Q2.13 — Which keyword quirks to raise upstream rather than copy
Candidates: the bare-OR side effect, the comma rule, stop-word-only searches returning everything.
(Feeds chunk 11.)

### Q2.14 — Pre-stem with ResourceSpace's own stemmer on both sides?
Core stems with `GetStem()` (`lib/stemming/<language>.php`: an irregular-plural table, then Porter)
at index time and query time; Typesense's Snowball stemmer agrees on 34 of 40 word-form pairs. If
the plugin stemmed every text value with `GetStem()` before indexing and every query word before
sending, with Typesense's own stemming off, the stems would be identical by construction, quoted
words would be sent unstemmed as core does, and phrases under stemming (Q2.5) would need no special
rule. Costs: a rebuild when the stemmer or `$stemming` changes (already true), stems visible in
Typesense (never shown — rows are hydrated from MySQL), and a check that Typesense's tokeniser
leaves the pre-stemmed tokens alone. Recommendation: yes, for SA12 and SA3.
**Direction from the acceptance rule (Q10.3, 2026-10-16):** stemmer drift changes sets, so the
stems must agree — pre-stem with core's stemmer, or decline when `$stemming` is on.

### Q2.15 — The full-text box: which operators, and how far
`"@FULL_TEXT:…"` is MySQL boolean mode over node names of every field, indexed or not; words without
an operator are ORed (A27 956 vs A28 9). **Decided 14 October: the rows are marked must** — the box
has to work. Typesense cannot OR words in `q`, so optional words need one search per alternative
merged client-side (SA8, shared with Q2.7). To decide: the operator subset (`+`, `-`, phrase, `*`,
bare words; the weighting operators dropped or vetoed), whether the merged set may differ in order
from MySQL's relevance, and whether non-indexed fields are in scope (Q1.1 option b) or the
difference for words found only there is accepted.

### Q2.16 — Short wildcards
A wildcard on a word under MySQL's full-text minimum length (`so*`, A14) is a `LIKE` on the keyword
table — indexed fields only — while a longer one (`sculpt*`, A11) reads node names of every field.
Replicate the split (prefix over indexed fields for short words, the all-fields rule for long
ones), or one rule for both and record the difference?

### Q2.17 — A wildcard inside a word
`sc*pture` reaches MySQL as `+sc*pture` and only the trailing part is honoured, so it behaves as
`sc*` (A13: 69,679). Replicate (treat it as `sc*`), veto (core then gives the same quirk), or serve
it properly where fragments allow (partial-indexed fields only)? Recommendation: veto.

**Catalogue marks (2026-10-20):** A13 is *check* — whether core really treats `sc*pture` as `sc*` is confirmed
before the mark is final; SA5 keeps the row.

### Q2.18 — Accent folding
MySQL's default collations fold accents and so does Typesense, but the local harness cannot check
it (SQLite). Accept Typesense's folding as parity on the strength of one live case per field type
(`café` / `cafe`), or add a MySQL-backed case to the live runner?

## Done when

Every keyword form in catalogue group A and the review's themes C and D has a decided behaviour,
the harness has a case for each, and the live A/B shows identical counts for the "copy core" set and
the accepted differences listed for the rest.
