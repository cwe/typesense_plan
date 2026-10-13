# F — Field, date, numeric and node forms; keyword semantics

F1–F8 are milestone M4 and plug into D4's field-token hand-off. F9 is milestone M6: the keyword
semantics that go beyond "words AND".

### F1 — Field lookup
- **Builds:** `include/search/fields.php` — short name → ref, type, `keywords_index`, `partial_index`,
  `active`, `field_constraint`, viewability; cached.
- **Size:** S · **Depends on:** B1
- **Must honour:** core matches the short name exactly (`basicyear` etc. are not fields); an unknown
  name makes the token two words; a non-viewable or inactive field is never searched (core hides
  inactive fields from positive matching); the lookup must *return* every flag the caller needs
  (lesson: an inert fix).
- **Blocking questions:** Q3.11
- **Proves it:** lookup battery.

### F2 — Text field:value
- **Builds:** in `keywords.php`: `field:value` on text fields.
- **Size:** S · **Depends on:** F1
- **Must honour:** the bare-colon operator is a tokenised word-contains filter honouring the field's
  tokeniser and stemming (`title:Gallery` 1,982 = a keyword search of that field); values with
  spaces or punctuation backtick-quoted; `word*` prefix within the field; multi-word AND;
  partial-indexed fields filter on `_p`; non-indexed fields per Q3.6; the candidate cap for
  `field:word*` (Q3.10).
- **Blocking questions:** Q3.6, Q3.10
- **Proves it:** catalogue group B rows.

### F3 — Fixed-list field:value
- **Builds:** the resolved/unresolved split for fixed-list fields.
- **Size:** S · **Depends on:** F1
- **Must honour:** core resolves an exact option name into `$node_bucket` before the hook (drop the
  token); when it does not — a word of an option, a wildcard, a category-tree child by name,
  dynamic-keyword text, a BOM-prefixed option name — the old plugin returned **everything**
  (`usagerights:social` 148 / 100,862). The rule is to veto (Q3.1) unless a best-effort match is
  chosen; `field:a;b` on a fixed list is resolved by core into two nodes.
- **Blocking questions:** Q3.1
- **Proves it:** catalogue group C rows, including the unresolved forms.

### F4 — Quoted field searches and the search bar's forms
- **Builds:** recognition of `"field:two words"`, `field:"x y"`, `subject:Birds` (rebuilt by the
  search bar from a clicked value).
- **Size:** S · **Depends on:** F2, F3
- **Must honour:** these are reached by clicking a metadata value and re-submitting; today they
  return nothing or everything.
- **Blocking questions:** Q3.2
- **Proves it:** catalogue group J rows.

### F5 — Date values
- **Builds:** `basicyear:` / `basicmonth:` / `basicday:` (the dropdowns) against `$date_field`;
  `date:<value>` exact on the `_q` representations; the legacy forms (`date:nnnn|05`, `date:202`,
  `date:05`) per decision.
- **Size:** S · **Depends on:** A4, F1
- **Must honour:** `date:2024` once matched the time `20:24` through the bare-colon operator → exact
  `:=` (P32); `basicyear:2024` 8,838 / 0 today; a year typed as a bare word matches dates in core
  (Q2.11).
- **Blocking questions:** Q3.3
- **Proves it:** catalogue group D single-value rows.

### F6 — Date ranges
- **Builds:** `field:rangestart<A>end<B>` → the range filter.
- **Size:** M · **Depends on:** A4, F1
- **Must honour:** core string-compares raw values (a year-only `2020-00-00` never matches a range
  starting after 1 Jan: +106 live) and accepts a non-real end (`2024-02-31`, EDTF `-99`: 822 /
  21,151 when the end is dropped); interval overlap on `_range_start/_end` is the alternative
  (P5); one value on a date-range field returns nothing in core and the exact search has reversed
  parameters (Core 2); non-indexed date fields per Q3.6.
- **Blocking questions:** Q3.4, Q3.5, Q3.6
- **Proves it:** catalogue group D range rows, both bounds, padded and EDTF ends, year-only values.

### F7 — Numeric fields
- **Builds:** `numrange<min>|<max>` → `_f`; plain value → `_q`; `neg` → `−`.
- **Size:** S · **Depends on:** F1, B1
- **Must honour:** both bounds → `[min..max]`; a single bound is an exact match in core (P6); plain
  `field:42` on a numeric field must use `_q` (filtering `_s` gave 0); `numrange` on a text field
  compares titles as numbers in core (Q3.7); live test field 170 (12 values) proved 8/8 cases and
  composition with keywords, sorts, restypes and `!last`.
- **Blocking questions:** Q3.7
- **Proves it:** catalogue group E rows and the field-170 cases.

### F8 — Node forms
- **Builds:** the node battery and the remaining node behaviours in `nodes.php`.
- **Size:** S · **Depends on:** D4
- **Must honour:** parity by construction (core resolves tokens before the hook, including NOT edge
  cases and the first-token-only view check); the filter-ops guard (P34); category-tree hierarchy
  via stored ancestors; the AND-logic toggle untested (needs a tree fixture); a node filter on a
  non-indexed field works (`nodes[]` from all nodes); relevance order of node-only searches is Q7.1.
- **Blocking questions:** Q3.9
- **Proves it:** node battery: `@@389@@!388`, `@@!!389`, mixed with keywords, nonexistent node,
  600/1,000-id buckets, 40/41-node chains, a parent tree node, the AND-logic toggle on both engines.

### F9 — Keyword semantics (milestone M6), one sub-task each
- **F9a Stop words** — strip `$noadd` words from `q`; a stop-word-only search and phrases containing
  stop words per Q2.4. Today `the sculpture` 49,468 / 17,228.
- **F9b Prefix, candidates and split/join** — `prefix=false` unless a wildcard; raised candidate caps
  for `word*` and `field:word*`; `split_join_tokens=off`; leading/middle wildcards per Q2.3.
- **F9c Phrases under stemming** — Q2.5; today `"sculpture park"` 6,883 / 0 with `$stemming` on
  because the index holds stems and phrase tokens are not stemmed.
- **F9d Negated phrase** — `-"…"` per Q2.6 (today dropped → everything).
- **F9e Punctuated tokens and commas** — Q2.8.
- **F9f Field-scoped OR** — `title:a;b` as two ORed filters, or keep the veto (Q2.7).
- **F9g Type names and usernames as words** — Q2.9 / Q1.7.
- **F9h Related keywords** — synonym sets kept by the hook, or veto (Q2.10); one pair on the test
  system (antony/anthony).
- **F9i Bare years and numbers** — Q2.11.
- **F9j Typo tolerance** — Q2.2.
- **Proves it:** catalogue group A rows and review themes C and D; the accepted-divergence list
  updated for every decided difference.
