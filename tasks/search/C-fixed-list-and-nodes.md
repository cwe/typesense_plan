# SC — Fixed-list fields and node tokens (catalogue group C)

An option of a checkbox, dropdown, radio, tree or dynamic-keyword field can be named
(`shortname:option`, quoted when it has spaces) or given as a node token (`@@nodeID`). Core resolves
an exact option name into a node **before the hook**; anything it cannot resolve becomes a keyword
search inside the field. Three packages: SC1 and SC2 are milestone M2 (parity by construction, but
the token must be handled), SC3 is M4.

### SC1 — Node tokens
- **Rows:** C13 C14 C15 C16 C17 C18 C19 C20 C21 C22 I10 H1 H4 H6 · cross: H17 (SD1), H18 (SC2), H28 (SF3), H29 (SF6), H13 (SA4)
- **Builds:** in `search/criteria.php` — `$node_bucket` → one `nodes:=[…]` clause per bucket,
  `$node_bucket_not` → `nodes:!=[…]`; the filter-operations guard.
- **Size:** S · **Depends on:** D3, B3 (`nodes[]` from all nodes) · **Milestone:** M2
- **Core:** `@@` tokens are removed from the string and resolved by `resolve_given_nodes()`
  before the hook: `@@a@@b` is one OR bucket (IN), separate tokens are AND buckets (one join
  each), `@@!a` is a NOT EXISTS; NOT is honoured only on a single token (`@@!389@@388` ORs both);
  an unknown node is kept and joins on a node no resource has (0); a comma between tokens is
  ignored; dynamic-keyword and large-field nodes are ordinary nodes.
- **Serve:** AND across buckets, OR within; `nodes:!=[a,b]` is any-of like core's NOT EXISTS;
  `nodes[]` holds every node of the resource regardless of index flags, so a node of a
  non-indexed field works; category-tree ancestors are stored on the resource at save, so a
  parent node matches its children's resources with nothing extra.
- **Must honour:** `filter_by` allows 100 operations and a GET 4,000 bytes (P34): 600- and
  1,000-id OR buckets are served, a non-`v` user's AND chain is served to 40 nodes and vetoed from
  41 (the restrictions cost 19), core itself fails at 58 joins; an over-limit join expression ANDed
  with other clauses returns 0 **silently** — the guard vetoes first;
  `$category_tree_search_use_and_logic` (one clause per node) is implemented but untested — the
  battery needs a tree; relevance order of a node-only search differs (D9, Q7.1).
- **Fixture:** the base fixture's tree (Animals → Birds, Mammals); generated 600/1,000-id buckets
  and a 41-node chain.
- **Battery:** `harness/ab/SC1_node_tokens.php` — every row above plus the harness node cases
  (`@@389@@!388`, `@@!!389`, mixed with keywords, the boundary buckets, a parent tree node, the
  AND-logic toggle on both engines).
- **Blocking questions:** Q3.9

### SC2 — Option names core resolves
- **Rows:** C1 C2 C6 C8 C10 C11 C23 C24 C26 H2 H18 J8 J9 J10
- **Builds:** in `search/criteria.php`: drop a `field:value` token (quoted or not) from `q` when core has
  already resolved it into `$node_bucket`.
- **Size:** S · **Depends on:** SC1, D2 · **Milestone:** M2
- **Core:** an exact option name (case-insensitive; quoted when it has spaces; `a;b` for two
  options; also on fields not flagged for indexing, C8) becomes a node in the bucket; when at least
  one `;` value is an exact option the other values are discarded (C26 = `usagerights:website`);
  the search page rewrites `materials:bronze;wood` into `@@6125@@6130` before searching (C24) and
  the advanced form posts ticked options as `@@` tokens, one OR token per field or one token per
  option with `$checkbox_and` (J8, J9), a dropdown as one token (J10).
- **Serve:** nothing to translate — the bucket is SC1's; the token must be dropped from `q`
  whether it is bare, quoted (`"usagerights:Social media"`, C2: 148 / 0 because the quoted token
  stayed in `q`) or an OR of names. The rule: a field token whose field is a fixed-list type and
  whose value core resolved is dropped; whether it was resolved is known from `$node_bucket`
  (compare the resolved node refs of the named field) — not by assuming.
- **Must honour:** the old plugin assumed every fixed-list token was resolved and so returned
  **everything** for the unresolved ones (SC3); `$node_bucket` also carries UI `@@` selections,
  so the comparison must be per field; the BOM case (C7) is unresolved in core because the stored
  name starts with an invisible character — P33 cleans index values but core's resolver does not
  (so C7 stays a keyword search in core).
- **Fixture:** the base fixture (country dropdown, keywords dynamic list, subject tree, a
  translated option `~en:Germany~fr:Allemagne`).
- **Battery:** `harness/ab/SC2_resolved_options.php` — the rows above, harness B22, P9 (a value
  given in another language), S2.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** an option name and a node token of the same field are two AND buckets (`ds:519-545`, `dsk:259-265`, `dsn:17-18`); option-name matching sees top-level nodes only (`nf:333`, `$recursive = false`), so a tree child is unreachable as `field:name` — 29,111 children on the second database; with `$checkbox_and` the form sends `word, , @@a, @@b` and the double comma is harmless (`ds:529-530`).
- **Blocking questions:** — (confirm the drop rule with Q3.1)

### SC3 — Option text core does not resolve
- **Rows:** C3 C4 C5 C7 C9 C12 B9 · cross: C25 (SA7), the tree child by name (SJ1)
- **Builds:** the fixed-list word search: `field_<ref>_ss:<word>` (bare-colon word-contains over
  the option names), with wildcard and multi-word forms.
- **Size:** M · **Depends on:** SB1, SC2, B2 (the `_ss` entry for every fixed-list field) · **Milestone:** M4
- **Core:** a value that is not an exact option name is a keyword search restricted to the field:
  `usagerights:social` matches the option words through `node_keyword` **if the field is flagged
  for indexing** (148); on a non-indexed fixed-list field the option words are not keywords, so
  nothing (C9: `source:digital` 0, although `"source:Digital Camera"` resolves); one word of a
  dynamic keyword (C12 `artistname:moore` 2,713) and of a dynamic field named with a slash (B9) the
  same; `usagerights:soc*` is a LIKE on keywords within the field (148); `usagerights:social media`
  binds social and leaves media as free text (148); an option stored with a leading byte-order
  mark is never an exact match (C7), so it is a keyword search too.
- **Serve:** `field_<ref>_ss:<word>` where `_ss` is the field's option names (cleaned, P33), for
  **indexed** fixed-list fields; for a non-indexed fixed-list field filter on a field that does not
  exist, which matches nothing (`validate_field_names=0`) — core's answer; a wildcard →
  `_ss:soc*`; a category-tree child by name (`subject:Birds`) resolves in core to the node when it
  is an exact name, so it is SC2, and otherwise this. This is parity, not best effort — the
  bare-colon operator is the same tokenised word match as SB1.
- **Must honour:** the old plugin returned **everything** for every row here (148 / 100,862;
  harness B23–B29, S8) — the worst class of divergence, and the reason principle 7 says
  everything-or-nothing is never accepted; `_ss` is already a `string[]` stemmed iff `$stemming`,
  so the word match works against it as it does against `_s` (verify the tokeniser treats array
  elements as separate values), and the
  BOM-stripped name in the index must still match core's keyword (core indexes the cleaned word
  too: `cleanse_string`).
- **Fixture:** a dropdown option with a space (`United Kingdom`), a dynamic keyword with two
  words (`modern art`), an option whose name starts with a BOM, an indexed and a non-indexed
  fixed-list field.
- **Battery:** `harness/ab/SC3_unresolved_options.php` — C3, C4, C5, C7, C9, C12, B9, harness
  B23–B29, S8.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** one word of an option or dynamic keyword falls back to a keyword search restricted to the field and needs the field indexed (`dsk:291-296`, `516`); a wildcard on an option (`field:soc*`) is a full-text prefix on option names restricted to the field, works on non-indexed fields, and a two-letter prefix falls to the LIKE branch; `field:a;b` silently drops an alternative that is not an option name once another matches (`dsk:264-268`); the test system's byte-order-mark option names are data, not a rule (none on the second database).
- **Blocking questions:** Q3.1, Q3.12, Q1.1
