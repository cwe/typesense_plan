# Core search catalogue — required cases

A copy of `plugins/typesense_search/docs/core-search-catalogue.md` (branch `typesense_search_catalogue`, commit
`0a67987c`, measured against the test database on 6–9 October 2026) for deciding which cases the rewrite must
serve. The old plugin's totals and the hand-check column are left out — they belong to the code being replaced.
**Core** is what core returned for the core user and is the parity target; **Package** is the work package that
owns the row in [tasks/search/coverage.md](tasks/search/coverage.md) (D-tasks are the pipeline).

## How to mark a row

Put one of these in the **Required** column:

- **must** — serve it, identical to core.
- **nice** — serve it if the mechanism is cheap, otherwise decline to core.
- **no** — never serve it; the plugin declines and core answers.
- **n/a** — unreachable, or not applicable on the systems this is for.
- **check** — core's behaviour is verified before the mark is set; the row keeps its package meanwhile. No row
  carries it since 2026-10-30: the 77 checks were read from core's code and confirmed on two databases
  ([tasks/search/checks-findings.md](tasks/search/checks-findings.md)) and the marks decided in
  [tasks/search/checks-review.md](tasks/search/checks-review.md); G42 became four rows there.
- **bug** — core's own result on this row is wrong (the Core numbers are the review's); whether the plugin
  copies, fixes or declines it is Q11.3, not this mark. The mark the row had before is kept in the note.

A note after the value is welcome (`must — the search bar sends this`). Rows the pages assemble (group J, the
rebuilt search-bar text, the simple-search dropdowns) reach users without typing. Node tokens, resolved options,
parameters and windows are parity by construction and cost nothing. Whether a core bug is copied, fixed or
declined is Q11.3, separate from this mark.

The test for a row whose core behaviour looks incidental (a string comparison, a MySQL quirk, a side effect of
the keyword table): **would the rule make sense on a database with different data?** If it only matters for a
shape this test database happens to have, mark it `nice` or `no` rather than `must` — the plugin declines it and
core answers, which matches core at no cost (principle 11 in [principles.md](principles.md)). The Core total is
evidence that the rule was understood, not the target: the rewrite is measured on the designed fixture and on a
second, differently shaped dataset, with live totals as a smoke test. When the marks are in, the planned disposition of every row in
`coverage.md` is updated from them, packages with no remaining rows are parked, and the worksheet is pruned.

Every search string is exactly what the API receives (`do_search`, or `search_get_previews` for a day limit);
parameters not shown are `restypes=""`, `order_by=relevance`, `archive=0`, `sort=desc`, `fetchrows=0,1`.
Workflow states 1, 2 and 3 are hidden from the test users (`z1 z2 z3`), so every total is over the states
they can see. Rows marked n/a in the Core column cannot be sent through the API; the local harness can still
run them (Q10.14).

## The test database

The examples use values that exist on the test database: its field short names, option names, node refs, collection refs
and user refs. `live/catalogue_values.php` lists the candidates on any database, so the cases can be rewritten for another one.

## Contents

- [A. Keywords](#a-keywords)
- [B. Field-specific terms (text fields)](#b-field-specific-terms-text-fields)
- [C. Fixed-list fields and node tokens](#c-fixed-list-fields-and-node-tokens)
- [D. Dates](#d-dates)
- [E. Numeric field](#e-numeric-field)
- [F. Special searches](#f-special-searches)
- [G. Parameters](#g-parameters)
- [H. Combinations](#h-combinations)
- [I. Syntax edge cases](#i-syntax-edge-cases)
- [J. Strings as the advanced search page assembles them](#j-strings-as-the-advanced-search-page-assembles-them)
- [K. What the indexing flags control](#k-what-the-indexing-flags-control)

## A. Keywords

Free text. Words are separated by spaces or commas and are ANDed; each word is looked up in the keyword table (stemmed when $stemming is on) and matched through node_keyword over every indexed field the user may see. A word that is not in the keyword table ends the search with a suggestion instead of results.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| A1 | One keyword | `sculpture` | Keyword resolved (stem), one union over node_keyword, scored by hit counts. | do_search_keywords.php:358, do_search_keywords.php:717 | [Searching](https://www.resourcespace.com/knowledge-base/user/searching-in-resourcespace) | 49,468 | SA1 | must |
| A2 | Two keywords | `sculpture landscape` | One union per word, all must match (AND). | do_search.php:139, do_search_union_assembly.php:55 | [Searching](https://www.resourcespace.com/knowledge-base/user/searching-in-resourcespace) | 34,408 | SA1 | must |
| A3 | Comma separated | `sculpture, landscape` | A comma is a separator like a space; same as A2. | search_functions.php:2054 | [Searching](https://www.resourcespace.com/knowledge-base/user/searching-in-resourcespace) | 34,408 | SA1 | must |
| A4 | Upper case | `SCULPTURE Park` | Lower-cased before matching. | search_functions.php:2157 | – | 19,680 | SA1 | must |
| A5 | Plural (stemming) | `sculptures` | Stemmed to the same keyword as A1. | do_search_keywords.php:358 | – | 49,468 | SA12 | must |
| A6 | Quoted single word | `"sculpture"` | Exact word, not stemmed, position join on one keyword. | do_search_keywords.php:758 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 49,069 | SA3 | must |
| A7 | Quoted phrase | `"sculpture park"` | Words at consecutive positions in one value of one field. | do_search_keywords.php:758, do_search_keywords.php:816 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 6,883 | SA3 | must |
| A8 | Phrase with a stop word | `"black and white"` | "and" is skipped and the position gap widened by one. | do_search_keywords.php:780 | – | 3,179 | SA3 | must |
| A9 | NOT a keyword | `-landscape` | Everything except resources with the keyword (NOT IN subquery). | do_search_keywords.php:303, do_search_keywords.php:471 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms), [Excluding terms](https://www.resourcespace.com/knowledge-base/user/excluding-search-terms) | 32,045 | SA4 | must |
| A10 | NOT a phrase | `-"sculpture park"` | Everything except resources with the phrase. | do_search_keywords.php:828 | – | 93,979 | SA4 | must — fixture: "park sculpture" is not excluded |
| A11 | Trailing wildcard | `sculpt*` | Full-text MATCH on node names ("+sculpt*") over all indexed fields. | do_search_keywords.php:617 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 51,007 | SA5 | must |
| A12 | Leading wildcard | `*scape` | LIKE '%scape' on the keyword table: only keywords of indexed fields and partial-index infixes; never a non-indexed value; matches the keyword's end, not a word boundary. | do_search_keywords.php:648 | – | 68,601 | SA5 | must — rule corrected: LIKE '%scape' on the keyword table, so only keywords of indexed fields and partial-index infixes; needs the reversed-keyword representation (SA5) |
| A13 | Wildcard inside a word | `sc*pture` | Goes to the full-text branch as "+sc*pture"; InnoDB reads "+sc*" as a required prefix and "pture" as an optional term, so this behaves like sc*. | do_search_keywords.php:617 | – | 69,679 | SA5 | no — a tokenizer accident: InnoDB honours the part before the *, so sc*pture = sc*; declined, core answers |
| A14 | Short wildcard (under 3 letters) | `so*` | LIKE "so%" on the keyword table instead of full text. | do_search_keywords.php:634 | – | 15,618 | SA5 | must |
| A15 | OR of two wildcards | `land*;sun*` | Semicolon forces the RLIKE branch: "(land.*?\|sun.*?)". | do_search_keywords.php:648, search_functions.php:3263 | – | 71,530 | SA7 | must — the single-token and field: forms; fixture must show the whitespace collapse |
| A16 | OR of two plain words | `sculpture;landscape` | The combined word "sculpture;landscape" is looked up as one keyword. First run: unknown, so a suggestion and no results. B5 then created that keyword as a side effect, and the re-run gave the OR (sculpture or landscape). See H32. | do_search_keywords.php:301, do_search_keywords.php:387 | – | 83,840 | SA7 | bug — Core 9 (was check); core looks up the literal keyword "sculpture;landscape": a suggestion on a fresh database; it was an OR on the test system only because B5 had created that keyword. A real OR would be an improvement over core (Q2.7) |
| A17 | Stop word alone | `the` | Skipped ($noadd); no terms left, so everything. | do_search_keywords.php:325, definitions.php:943 | – | 100,862 | SA2 | must — core drops the stop word and returns everything; case rule copied exactly (The ≠ the) |
| A18 | Stop word plus a word | `the sculpture` | Stop word dropped; same as A1. | do_search_keywords.php:325 | – | 49,468 | SA2 | must |
| A19 | Hyphenated word | `black-and-white` | Separators split it into black, and, white; then as A8 but unordered (AND of the words). | do_search_keywords.php:367, config.default.php:1340 | – | 3,179 | SA6 | must |
| A20 | Apostrophe | `atkinson's` | The apostrophe is a separator: atkinson AND s. | do_search_keywords.php:367 | – | 2 | SA6 | must |
| A21 | Related keyword | `antony` | Also matches the related keyword anthony (keyword_related). | do_search_keywords.php:436 | – | 1,491 | SA10 | must |
| A22 | The related word itself | `anthony` | Also matches antony; compare with A21. | do_search_keywords.php:436 | – | 1,491 | SA10 | must |
| A23 | Resource type name | `document` | Every resource of the type named Document, plus any with the word. | do_search_keywords.php:56 | – | 490 | SA9 | must |
| A24 | Contributor name | `admin` | With $index_contributed_by on, every upload by a user whose name matches; otherwise just the word (the option is off on the test system). | do_search_keywords.php:62, config.default.php:2284 | – | 13 | SA9 | must |
| A25 | A resource number | `416` | With $config_search_for_number off (default): the word 416 plus resource 416, which is ordered first. On: same as !resource416. | do_search.php:117, do_search.php:320, do_search.php:283 | [Searching](https://www.resourcespace.com/knowledge-base/user/searching-in-resourcespace), [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 100 | SA9 | must |
| A26 | A year as a word | `2024` | Dates are indexed as year, year-month and full date, so this matches resources dated 2024 in any date field, plus text containing 2024. | search_functions.php:2059 | – | 10,289 | SA9 | must |
| A27 | Full-text box | `"@FULL_TEXT:distant island"` | MATCH AGAINST in boolean mode over node names (what the advanced search full-text box sends). | do_search_keywords.php:30, search_functions.php:242 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search), [Full text searching](https://www.resourcespace.com/knowledge-base/user/fulltext-search) | 956 | SA8 | must — boolean mode: words without an operator mean any word (956), not all (9); optional words need one search per alternative, merged (SA8); words found only in non-indexed fields need Q1.1 option b |
| A28 | Full-text with operators | `"@FULL_TEXT:+distant +island"` | Boolean operators pass straight to MySQL. | do_search_keywords.php:30 | [Full text searching](https://www.resourcespace.com/knowledge-base/user/fulltext-search) | 9 | SA8 | must — required words map directly (SA8) |
| A29 | Full-text phrase | `"@FULL_TEXT:[QUOTES]distant island[QUOTES]"` | [QUOTES] becomes a double quote: a MySQL phrase search. | do_search_keywords.php:31 | – | 8 | SA8 | must — a phrase (SA8, SA3) |
| A30 | Misspelt word | `sculptre` | An unknown word gives no results and a sound-alike suggestion instead of an array; on the test system this misspelling happens to exist as a keyword, so A31 shows the real case. | do_search_keywords.php:395, do_search_suggest.php:4 | [Searching](https://www.resourcespace.com/knowledge-base/user/searching-in-resourcespace), [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 1 | SA11 | no — an unknown word is answered by core before the plugin is asked; declined |
| A31 | Known plus unknown word | `sculpture zzzzqqq` | No results; the suggestion drops the unknown word. | do_search_suggest.php:10 | [Searching](https://www.resourcespace.com/knowledge-base/user/searching-in-resourcespace) | 0 | SA11 | must — already so: core returns the "did you mean" suggestion before the plugin is consulted; results for the known word would need a core change (Q2.12) |
| A32 | Only an asterisk | `*` | Treated as the empty search. | do_search.php:85 | – | 100,862 | SA1 | must |
| A33 | Empty search | (empty) order_by=resourceid | No keyword unions: every resource the filters allow. | do_search.php:412 | – | 100,862 | SA1 | must |
| A34 | Full-text exclusion | `"@FULL_TEXT:+distant -island"` | The documented operators: + requires a word, - excludes one (compare A28). Run on 9 October, when the plugin group was returning core's own results for every form, so the plugin total here is not comparable with the 6 October rows. | do_search_keywords.php:30 | [Full text searching](https://www.resourcespace.com/knowledge-base/user/fulltext-search) | 880 | SA8 | must — required and excluded words map directly (SA8) |
## B. Field-specific terms (text fields)

shortname:value binds one word to one field. Only the word next to the colon is bound; the rest of a multi-word value is free text, unless the whole term is quoted as "shortname:two words", which the advanced search and the search bar produce.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| B1 | Word in a field | `title:sculpture` | Keyword union restricted to nodes of that field. | do_search_keywords.php:95, do_search_keywords.php:717 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 4,226 | SB1 | must |
| B2 | Two words after a field | `title:henry moore` | henry is bound to title; moore is free text. | search_functions.php:2103 | – | 251 | SB2 | must |
| B3 | Quoted phrase in a field | `"title:henry moore"` | Phrase search restricted to the field (the form the advanced search sends). | do_search_keywords.php:758, search_functions.php:275 | – | 251 | SB2 | must — fixture "Moore, Henry": B2 matches, B3 must not |
| B4 | Quotes after the colon | `title:"henry moore"` | Not the supported form: the tokens are title:"henry and moore", but the stray quotes are stripped, so it works out the same as B2. | search_functions.php:2103 | – | 251 | SB2 | no — an accidental parse (re-quote plus swallow); declined, core answers |
| B5 | OR inside a field | `title:sculpture;landscape` | Either word in the field (alternative keywords IN). | do_search_keywords.php:301 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 5,120 | SA7 | must |
| B6 | Wildcard in a field | `title:sculpt*` | Full-text MATCH restricted to the field. | do_search_keywords.php:617 | – | 4,522 | SB3 | must |
| B7 | Leading wildcard in a field | `title:*scape` | LIKE '%scape' on the keyword table, restricted to the field. | do_search_keywords.php:648 | – | 921 | SB3 | must — rule corrected: LIKE '%scape' on keywords restricted to the field (camera:*ikon = 0 live proves it) |
| B8 | Another text field | `credit:wilde` | As B1 on the credit field. | do_search_keywords.php:95 | – | 18,879 | SB1 | must |
| B9 | Field name with a slash | `photographer/videographer:wilde` | The name is looked up whole; one word of a dynamic-keyword option matches through node_keyword. | do_search_keywords.php:95, do_search_keywords.php:253 | – | 33,517 | SC3 | must |
| B10 | Punctuation in a field value | `exhibitiontitle:fabric-ation` | Core quotes the term and searches the phrase "fabric ation" in that field. | do_search_keywords.php:286 | – | 1,453 | SB4 | must |
| B11 | Dots in a field value | `dimensions:16.5x16.5cm` | Quoted, then split on the dots: the phrase 16, 5x16, 5cm. | do_search_keywords.php:286 | – | 20 | SB4 | must |
| B12 | Text field not flagged for indexing | `camera:nikon` | The field exists but has no keywords: nothing, although the values are there. | do_search_keywords.php:297 | – | 0 | SK1 | must |
| B13 | Multi-line field not flagged for indexing | `accessibilityalttext:island` | Nothing, as B12. | do_search_keywords.php:297 | – | 0 | SK1 | must |
| B14 | Partially indexed field, fragment | `originalfilename:ksho` | Fragments of 3+ letters are indexed, so "ksho" finds filenames containing "workshop". | search_functions.php:2270 | [Indexing metadata](https://www.resourcespace.com/knowledge-base/resourceadmin/indexing-metadata) | 665 | SB6 | must |
| B15 | Unknown field name | `foo:sculpture` | The colon becomes a space: foo and sculpture as two words (foo happens to exist as a keyword on the test system; an unknown word would give a suggestion). | do_search_keywords.php:112 | – | 25 | SB1 | must |
| B16 | NOT before a field term | `-title:sculpture` | Not supported: read as -title plus sculpture. | do_search_keywords.php:112 | – | 49,241 | SB5 | no |
| B17 | NOT after the colon | `title:-sculpture` | Quoted, then a NOT against the junk keyword "-sculpture" that resolve_keyword() creates: excludes nothing. | do_search_keywords.php:286, do_search_keywords.php:769 | – | 100,862 | SB5 | bug — NOT inside a field term excludes nothing and creates a junk keyword (I4, found 2026-10-22); was check; declined meanwhile |
| B18 | Stop word in a field | `title:the` | Skipped: everything. | do_search_keywords.php:325 | – | 100,862 | SB5 | must |
| B19 | Field hidden from advanced search, not indexed | `yearai:2025` | Nothing (no keywords); the field is still accepted by name. | do_search_keywords.php:297 | – | 0 | SK1 | must |
| B20 | Two terms, one field | `title:sculpture, title:park` | AND of two field-restricted unions. | do_search_union_assembly.php:55 | – | 580 | SB1 | must |
| B21 | Two fields | `title:sculpture, credit:wilde` | AND across fields. | do_search_union_assembly.php:55 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 316 | SB1 | must |
## C. Fixed-list fields and node tokens

An option of a checkbox, dropdown, radio, tree or dynamic-keyword field can be named (shortname:option, quoted when it has spaces) or given as a node token (@@nodeID). Named options become nodes; one word of an option falls back to a keyword search inside the field. Tokens: @@a@@b is OR, separate tokens are AND, @@!a is NOT.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| C1 | Option by name | `orientation:landscape` | Converted to the node (OR bucket of one). | do_search_keywords.php:253 | – | 58,028 | SC2 | must |
| C2 | Multi-word option, quoted | `"usagerights:Social media"` | Quoted field term resolved to the node; what the search bar rebuilds from a node. | do_search_keywords.php:253, searchbar.php:141 | – | 148 | SC2 | must |
| C3 | Multi-word option, unquoted | `usagerights:social media` | social is a word in the field; media is free text. | search_functions.php:2103 | – | 148 | SC3 | must |
| C4 | One word of an option | `usagerights:social` | Not an option name, so a keyword search restricted to the field. | do_search_keywords.php:253, do_search_keywords.php:297 | – | 148 | SC3 | must — fixture with two options sharing a word |
| C5 | Wildcard on an option | `usagerights:soc*` | Full-text prefix on option names restricted to the field (a two-letter wildcard takes the keyword LIKE branch instead). | do_search_keywords.php:617 | – | 148 | SC3 | must — rule corrected: full-text prefix on option names restricted to the field (2-letter wildcards take the keyword LIKE branch); works on non-indexed fields |
| C6 | OR of options | `materials:bronze;wood` | Both nodes in one OR bucket. | do_search_keywords.php:262 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 128 | SC2 | must |
| C7 | AND of options | `materials:bronze, materials:wood` | Two buckets: resources with both (the $checkbox_and form). On the test system the first option's name is stored with a leading byte-order mark, so it is not matched as an option and becomes a keyword search within the field; the plugin drops it instead (B3 in the review). | search_functions.php:293, do_search_nodes.php:17 | – | 0 | SC3 | must — fixture with clean names and resources holding one, both, neither |
| C8 | Dropdown option, quoted | `"source:Digital Camera"` | Resolved to the node by name although the field is not flagged for indexing. | do_search_keywords.php:253 | – | 2,224 | SC2 | must |
| C9 | One word of a dropdown option | `source:digital` | Keyword search in a fixed-list field not flagged for indexing: its option words are not in node_keyword, so nothing. | do_search_keywords.php:297 | – | 0 | SC3 | must |
| C10 | Checkbox option, one word | `yspactivity:learning` | Option name matched case-insensitively. | do_search_keywords.php:262 | – | 11,671 | SC2 | must |
| C11 | Dynamic keyword option, quoted | `"artistname:Henry Moore"` | Resolved to the node by full name. | do_search_keywords.php:253 | – | 2,713 | SC2 | must |
| C12 | One word of a dynamic keyword | `artistname:moore` | Keyword search in the field. | do_search_keywords.php:297 | – | 2,713 | SC3 | must — fixture with two options sharing a word |
| C13 | Node token | `@@389` | Removed from the string and joined as a node bucket. | do_search.php:111, do_search.php:519, do_search_nodes.php:17 | – | 58,028 | SC1 | must |
| C14 | OR of nodes | `@@389@@388` | One bucket with both nodes (IN). | do_search.php:519 | – | 94,936 | SC1 | must |
| C15 | AND of nodes | `@@425, @@389` | Two buckets, two joins. | do_search_nodes.php:17 | – | 49 | SC1 | must |
| C16 | AND of nodes, space separated | `@@389 @@425` | Same as C15. | do_search.php:519 | – | 49 | SC1 | must |
| C17 | NOT a node | `@@!432` | NOT EXISTS on resource_node. | do_search_nodes.php:31 | – | 100,461 | SC1 | must |
| C18 | NOT inside an OR word | `@@!389@@388` | NOT is only honoured on a single token; here it is ignored and both nodes are ORed. | do_search.php:538 | – | 94,936 | SC1 | must |
| C19 | Unknown node | `@@999999999` | The node has no field; the token is kept and joins on a node that no resource has. | do_search.php:534 | – | 0 | SC1 | must |
| C20 | Simple-search-bar checkbox | `@@17913` | The fixed-list field shown on the simple search bar. | searchbar.php:53 | [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 264 | SC1 | must |
| C21 | Dynamic keyword node | `@@2997` | The node of a dynamic-keyword option. | do_search_nodes.php:17 | – | 33,333 | SC1 | must |
| C22 | GPT keyword node | `@@460` | A node from a large dynamic-keyword field. | do_search_nodes.php:17 | – | 37,603 | SC1 | must |
| C23 | Option name and a node of the same field | `orientation:landscape, @@388` | Two buckets on one field: both options needed. | do_search_nodes.php:17 | – | 0 | SC2 | must — fixture confirming node 388's field |
| C24 | Option names as the search page converts them | `materials:bronze;wood` | Same as C6. pages/search.php rewrites this to @@6125@@6130 before searching; the API gets the text form (C6). | search.php:76, search.php:150 | – | 128 | SC2 | must |
| C25 | Documented OR shape with a multi-word value | `usagerights:social media;website` | The knowledge base gives country:united kingdom;spain. Only the first word is bound to the field and "media;website" becomes a bare OR, which is a suggestion (A16): nothing. Run on 9 October, when the plugin group was returning core's own results for every form, so the plugin total here is not comparable with the 6 October rows. | search_functions.php:2103, do_search_keywords.php:387 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 0 | SA7 | must |
| C26 | OR of an option name and a word | `usagerights:website;social` | When at least one value is an exact option, the others are discarded rather than searched as words: the same as usagerights:website alone (compare C4). Run on 9 October, when the plugin group was returning core's own results for every form, so the plugin total here is not comparable with the 6 October rows. | do_search_keywords.php:262 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 82 | SC2 | must |
## D. Dates

A date field (date, date and time, expiry, date range) accepts a value or a range after the colon. Values are matched as a prefix on the stored string, so a year, a month or a day all work; ranges compare strings. basicyear, basicmonth and basicday address $date_field from the simple search bar. These searches are slow in core on the test system (several seconds each).

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| D1 | Year | `date:2024` | LIKE "2024%" on the field's node names. | do_search_keywords.php:130, do_search_keywords.php:157 | – | 8,838 | SD1 | must |
| D2 | Year and month | `date:2024-05` | LIKE "2024-05%". | do_search_keywords.php:157 | – | 572 | SD1 | must |
| D3 | Full date | `date:2024-05-31` | LIKE "2024-05-31%". | do_search_keywords.php:157 | – | 317 | SD1 | must |
| D4 | Date stored with a time | `date:2017-05-10` | The prefix still matches a value stored with a time of day. | do_search_keywords.php:157 | – | 626 | SD1 | must — fixture with a timed value |
| D5 | Range | `date:rangestart2024-01-01end2024-02-29` | name >= start AND name <= "end 23:59:59". | do_search_keywords.php:180 | – | 822 | SD3 | must — Q3.4 (a) reproduces it; fixture with a -00 partial and a timed end-day value |
| D6 | Range, start only | `date:rangestart2026-01-01` | name >= start. | do_search_keywords.php:193 | – | 2,006 | SD3 | must |
| D7 | Range, end only | `date:rangeend2005-12-31` | name <= end. | do_search_keywords.php:198 | – | 722 | SD3 | must — plus a row for the stale-bound quirk (date:rangestart…, date:rangeend…) |
| D8 | Range as the EDTF box sends it | `date:rangestart2024-00-00end2024-12-99` | "2024/2024" in the EDTF box becomes this; string comparison still brackets the year. | search_functions.php:3341 | – | 8,838 | SD3 | must — fixture with a 2024-12-31 23:xx value |
| D9 | Range, end month without a day | `date:rangestart2024-01-01end2024-02-31` | The form pads a missing day with 31; as a string "2024-02-31" is after every February date. | search_functions.php:3362 | – | 822 | SD3 | must |
| D10 | Simple search year | `basicyear:2024` | Joins $date_field once and filters LIKE "2024-__-__%". | do_search_keywords.php:163, do_search_keywords.php:851 | [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 8,838 | SD2 | must |
| D11 | Simple search month, any year | `basicmonth:05` | LIKE "____-05-__%". | do_search_keywords.php:851 | [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 9,511 | SD2 | must |
| D12 | Simple search day, any month | `basicday:05` | LIKE "____-__-05%" (the day dropdown needs $searchbyday). | do_search_keywords.php:851, config.default.php:539 | – | 3,227 | SD2 | must |
| D13 | Year and month dropdowns | `basicyear:2025, basicmonth:06` | One join, LIKE "2025-06-__%". | do_search_keywords.php:851 | [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 1,542 | SD2 | must |
| D14 | Year, month and day dropdowns | `basicyear:2025, basicmonth:06, basicday:05` | LIKE "2025-06-05%". | do_search_keywords.php:851 | [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 256 | SD2 | must |
| D15 | Legacy "any year" syntax | `date:nnnn\|05` | n becomes _, \| becomes -: LIKE "____-05%". | do_search_keywords.php:146 | – | 9,511 | SD2 | must |
| D16 | startdate / enddate form inputs | `startdate:2024-01-01, enddate:2024-12-31` | search_form_to_search_query still assembles these, but no page posts the inputs and nothing in do_search handles them: unknown field names, so words and a suggestion. | search_functions.php:188, do_search_keywords.php:112 | – | 0 | SD2 | no — dead inputs; core answers before the hook |
| D17 | Date-and-time field, full date | `expirydate:2029-05-31` | As D3 on a type 4 field. | do_search_keywords.php:157 | – | 224 | SD1 | must |
| D18 | Date-and-time field, year | `expirydate:2029` | As D1. | do_search_keywords.php:157 | – | 3,171 | SD1 | must |
| D19 | Date-and-time field, range | `expirydate:rangestart2029-01-01end2029-12-31` | As D5. | do_search_keywords.php:180 | – | 3,171 | SD3 | must |
| D20 | Partial value (year only) | `dateofartwork:2025` | Prefix match includes year-only values stored as YYYY-00-00. | do_search_keywords.php:157 | – | 2,593 | SD1 | must |
| D21 | Month, including month-only values | `eventdate:2022-04` | Prefix match includes month-only values. | do_search_keywords.php:157 | – | 509 | SD1 | must |
| D22 | Date field not flagged for indexing | `datephototaken:2024` | Date terms join the node table directly, so the index flag does not matter. | do_search_keywords.php:130 | – | 3,676 | SD5 | must — fixture only |
| D23 | Same field, full date | `datephototaken:2012-01-26` | Matches values stored with a time of day. | do_search_keywords.php:157 | – | 4,549 | SD5 | must |
| D24 | Date-range field, one value | `daterangetest:2020` | Meant to find ranges containing 2020; the parameters are bound in the wrong order, so nothing (Core 2 in the review). | do_search_keywords.php:149 | – | 0 | SD4 | bug — Core 2 (was check) |
| D25 | Date-range field, range | `daterangetest:rangestart2020-01-01end2021-12-31` | Overlap, not containment: some value of the field >= start AND some value <= end, as strings (the two nodes of a range are not told apart). | do_search_keywords.php:180 | – | 1 | SD4 | must — rule corrected to overlap: some value >= start AND some value <= end; two _k values per resource (SD4 schema addition); fixture: partial overlap, enclosing, outside, open-ended |
| D26 | Two values for one date field | `date:2024, date:2025` | Two joins, both must match: only resources holding both years. | do_search_keywords.php:157 | – | 0 | SD1 | must |
| D27 | OR on a date | `date:2024;2025` | No OR for dates: LIKE "2024;2025%", nothing. | do_search_keywords.php:157 | – | 0 | SD1 | no — no form emits it; declining gives core's empty answer |
| D28 | Another date-and-time field | `copyrightexpirationd:2030` | As D1. | do_search_keywords.php:157 | – | 47 | SD1 | must |
## E. Numeric field

A text field with field_constraint = 1 takes shortname:numrangeMIN|MAX. "neg" stands for a minus sign. One bound means an exact value.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| E1 | Range | `numberfield:numrange0\|100` | name >= 0 AND name <= 100 (numeric compare). | do_search_keywords.php:206 | – | 9 | SE1 | must — worked with the previous code |
| E2 | Exact (one bound) | `numberfield:numrange42\|` | name = 42. | do_search_keywords.php:222 | – | 2 | SE1 | must — worked with the previous code |
| E3 | Max only | `numberfield:numrange\|100` | Also exact: name = 100. | do_search_keywords.php:222 | – | 1 | SE1 | must — worked with the previous code |
| E4 | Negative bounds | `numberfield:numrangeneg50\|neg1` | -50 to -1. | do_search_keywords.php:213 | – | 2 | SE1 | must — worked with the previous code |
| E5 | Decimal bounds | `numberfield:numrange3\|3.14` | 3 to 3.14. | do_search_keywords.php:206 | – | 1 | SE1 | must — worked with the previous code |
| E6 | Plain number in the field | `numberfield:42` | Not a range: an ordinary keyword search for "42" in the field. | do_search_keywords.php:297 | – | 2 | SE1 | must — it is SB1; fixture with 42.5 and 042 |
| E7 | numrange on a text field | `title:numrange1\|10` | Accepted for any type 0 field: compares title values as numbers. | do_search_keywords.php:206 | – | 2,937 | SE1 | no — the form emits numrange only for field_constraint = 1 fields; incidental casting; declined |
## F. Special searches

A search starting with ! is handled by search_special() (or by the keyword stage for !empty). Anything after the first space is keywords, nodes or field terms applied on top. A second word starting with ! is ignored. Resource types still apply to most of them, workflow states do not apply to collections, lists and the pending searches, and the z permissions always apply.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| F1 | !last | `!last100` | The 100 highest refs that pass the filters. | search_functions.php:1110 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 100 | SF1 | must |
| F2 | !last, first rows | `!last5` fetchrows=0,5 | Relevance order becomes ref DESC. | search_functions.php:1115 | – | 5 [127607,127608,127609,127610,127611] | SF1 | must |
| F3 | !last with no number | `!lastabc` | Not an integer: 1000. | search_functions.php:1134 | – | 1,000 | SF1 | must |
| F4 | !last plus a keyword | `!last100 sculpture` | The keyword applies, but "100 sculpture" is read as the number: not an integer, so !last1000 (F69 has the comma form that keeps 100). | do_search.php:126, search_functions.php:1141 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 1,000 | SF1 | bug — Core 7 (was must) |
| F5 | !last plus a node | `!last1000 @@389` | Node join plus the aggregate hit count without GROUP BY: one row (Core 1 in the review). | search_functions.php:1141, do_search_nodes.php:24 | – | 1 | SF1 | bug — Core 1 (was must) |
| F6 | !nodownloads | `!nodownloads` | Not in daily_stat as a download. | search_functions.php:1143 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 94,493 | SF17 | no |
| F7 | !duplicates (all) | `!duplicates` | Every resource whose checksum appears more than once. | search_functions.php:1151 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 59 | SF17 | no |
| F8 | !duplicates of one resource | `!duplicates123007` | Resources with the same checksum, including itself. | search_functions.php:1160 | – | 2 | SF17 | no |
| F9 | !collection | `!collection3808` | Members of a readable collection, any workflow state. | search_functions.php:1191, search_functions.php:825 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 55 | SF2 | must |
| F10 | !collection, public | `!collection3618` | Public collections are readable by everyone. | search_functions.php:1209 | – | 949 | SF2 | must |
| F11 | !collection, featured | `!collection3706` | Featured collections pass through featured_collection_check_access_control. | search_functions.php:1231 | – | 262 | SF2 | must |
| F12 | !collection, featured but not public | `!collection3125` | Access decided by the featured-collection rules. | search_functions.php:1231 | – | 257 | SF2 | must |
| F13 | !collection, someone else's private | `!collection158` | Not in the user's readable set: empty. | search_functions.php:1235 | – | 0 | SF2 | must |
| F14 | !collection, own empty collection | `!collection{owncollection}` | Readable, no members. | search_functions.php:1209 | – | 0 | SF2 | must |
| F15 | !collection, own upload collection | `!collection-{userref}` | A negative ref is the user's upload collection. | search_functions.php:1229 | – | 0 | SF2 | must |
| F16 | !collection plus a keyword | `!collection3808 sculpture` | Keyword applied inside the collection. | do_search.php:126 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 55 | SF2 | must |
| F17 | !collection, comma then keyword | `!collection3808, sculpture` | Same as F16 (the comma before the space is dropped). | do_search.php:126 | – | 55 | SF2 | must |
| F18 | !collection plus a node | `!collection3808 @@389` | Node join inside the collection. | do_search.php:111 | – | 1 | SF2 | must |
| F19 | !collection in collection order | `!collection3808` order_by=collection, sort=asc, fetchrows=0,5 | c.sortorder, then date added, then ref. | search_functions.php:3206 | – | 55 [59367,59354,59348,59320,59315] | SF2 | must |
| F20 | !collection with a resource type | `!collection3808` restypes=5 | Resource types are not applied to collection searches. | search_functions.php:751 | – | 55 | SF2 | must |
| F21 | !collection with a workflow state | `!collection3808` archive=2 | Workflow states are not applied to collections; the z permissions still are. | search_functions.php:825, search_functions.php:873 | – | 55 | SF2 | must |
| F22 | !related | `!related121409` | Both directions of resource_related; all workflow states. | search_functions.php:1336, search_functions.php:831 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 488 | SF11 | must |
| F23 | !related plus a keyword | `!related121409 sculpture` | Keyword applied to the related set. On the test system this query ran past the 120-second API timeout. | do_search.php:126 | – | error: HTTP 0 connection failed: Operation timed out after 120002 m | SF11 | must |
| F24 | !relatedpushed | `!relatedpushed121409` | Related resources of types with push_metadata (view page). | search_functions.php:1295 | – | 0 | SF11 | must |
| F25 | !geo | `!geo53p5bm1p7t53p7bm1p4` | Bounding box: m is minus, p is the point, b separates lat/long, t separates the corners (53.5,-1.7 to 53.7,-1.4). | search_functions.php:1355 | [Geographic search](https://www.resourcespace.com/knowledge-base/user/geographic-search) | 4,893 | SF14 | no |
| F26 | !colourkey | `!colourkeyEPNB` | First four letters of the colour key, resources with a preview. | search_functions.php:1373 | – | 1,979 | SF15 | no |
| F27 | !colour | `!colourE` | Colour key LIKE "E%" OR "_E%". | search_functions.php:1382 | – | 70,574 | SF15 | no |
| F28 | !rgb | `!rgb:610,391,405` | Nearest average colour, at most 500 rows. | search_functions.php:1398, search_functions.php:1407 | – | 500 | SF17 | no |
| F29 | !nopreview | `!nopreview` | has_image = 0. | search_functions.php:1410 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 105 | SF12 | must |
| F30 | !resource | `!resource416` | One resource by ref (the search bar resource ID box). | search_functions.php:1421, search.php:236 | – | 1 | SF4 | must |
| F31 | !resource plus a term with digits | `!resource416, date:2024` | Every digit in the string forms the ref: resource 4162024, nothing (Core 3 in the review). | search_functions.php:1422 | – | 0 | SF4 | bug — Core 3 (was check) |
| F32 | !archivepending | `!archivepending` | State 1 (pending review); hidden from these users by z1. | search_functions.php:1426, search_functions.php:873 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 0 | SF7 | must |
| F33 | !userpending | `!userpending` | State -1; non-admins only see their own. | search_functions.php:1430, search_functions.php:868 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 0 | SF7 | must |
| F34 | !contributions | `!contributions9` | created_by = 9. | search_functions.php:1438 | – | 41,283 | SF5 | must |
| F35 | !contributions, own | `!contributions{userref}` | Own uploads ($open_access_for_contributor lifts the access filter). | search_functions.php:1447 | – | 0 | SF5 | must |
| F36 | !images | `!images` | has_image > 0. | search_functions.php:1459 | – | 100,757 | SF12 | must |
| F37 | !unused | `!unused` | In no collection. | search_functions.php:1464 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 6,967 | SF16 | must — if memberships of every collection kind are indexed (Q5.7); core counts private, selection, upload, share-upload and request collections as used |
| F38 | !list | `!list416:4460:8692` | Refs separated by colons (the example refs are in three different states); workflow states not applied, z permissions are. | search_functions.php:1480, search_functions.php:825 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 1 | SF3 | must |
| F39 | !listall | `!listall416:4460:8692` | Same, by name "all"; the filter difference is in search_filter, which treats both alike. | search_functions.php:1484 | – | 1 | SF3 | must — treated as !list |
| F40 | !list plus a keyword | `!list416:4460:8692 sculpture` | Keyword applied to the list. | do_search.php:126 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 1 | SF3 | no |
| F41 | !list with commas | `!list416,4460` | Only the part before the first comma is the list: resource 416. | search_functions.php:1489 | – | 1 | SF3 | no |
| F42 | !hasdata | `!hasdata170` | Resources with any value in the field (by ref only); all workflow states. | search_functions.php:1512, search_functions.php:831 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 12 | SF6 | must |
| F43 | !hasdata on a non-indexed field | `!hasdata93` | Node join, so indexing does not matter. | search_functions.php:1512 | – | 63,316 | SF6 | must |
| F44 | !hasdata plus a keyword | `!hasdata170 sculpture` | Keyword applied. | do_search.php:126 | – | 0 | SF6 | no |
| F45 | !empty by ref | `!empty170` | Handled as a keyword: resources with no value in the field. | do_search_keywords.php:310, do_search_keywords.php:570 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 100,850 | SF10 | must |
| F46 | !empty by short name | `!emptynumberfield` | Meant to be F45 by name, but the name is bound as an integer (type "i"), which MySQL reads as 0 and matches the first field whose name is not a number: field 1, keywords. | do_search_keywords.php:316 | – | 100,004 | SF10 | bug — Core 5 (was no) |
| F47 | !empty on a field of one type | `!empty95` | Meant to limit candidates to the field's resource types, but the type condition sits inside the NOT IN subquery, so resources of every other type come back too. | do_search_keywords.php:566 | – | 100,457 | SF10 | bug — Core 6 (was must) |
| F48 | !empty with a resource type | `!empty170` restypes=1 | Candidates limited to the given types. | do_search_keywords.php:555 | – | 93,054 | SF10 | must — type filter by construction (field 170 is global) |
| F49 | !empty, unknown field | `!emptynosuchfield` | Meant to exit with "invalid !empty search"; because of the integer binding in F46 it becomes !empty1 (keywords) instead. | do_search_keywords.php:316, do_search_keywords.php:320 | – | 100,004 | SF10 | bug — Core 5 (was no) |
| F50 | !properties file extension | `!propertiesfext:jpg` | file_extension LIKE "jpg". | search_functions.php:1524, search_functions.php:1566 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 66,810 | SF9 | must |
| F51 | !properties, not an extension | `!propertiesfext:-jpg` | NOT LIKE. | search_functions.php:1569 | – | 34,052 | SF9 | must |
| F52 | !properties, extension wildcard | `!propertiesfext:jp*` | * becomes %: jpg and jpeg. | search_functions.php:1567 | – | 71,886 | SF9 | must — expectation written as the LIKE |
| F53 | !properties minimum height | `!propertieshmin:3000` | rdim.height >= 3000. | search_functions.php:1541 | – | 69,894 | SF9 | must |
| F54 | !properties, several | `!propertieswmin:4000;hmin:3000` | Semicolon-separated properties are ANDed. | search_functions.php:1528 | – | 55,705 | SF9 | must |
| F55 | !properties minimum size | `!propertiesfmin:5` | file_size >= 5 MB. | search_functions.php:1558 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 53,138 | SF9 | must |
| F56 | !properties maximum size | `!propertiesfmax:1` | file_size <= 1 MB. | search_functions.php:1563 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 11,993 | SF9 | must |
| F57 | !properties has preview | `!propertiespi:1` | has_image = 1. | search_functions.php:1580 | – | 100,072 | SF9 | must |
| F58 | !properties contributor | `!propertiescu:9` | created_by = 9. | search_functions.php:1584 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 41,283 | SF9 | must |
| F59 | !properties orientation | `!propertiesorientation:portrait` | Compares resource_dimensions height and width. | search_functions.php:1588 | – | 36,092 | SF9 | must — fixture with a dimension-less resource |
| F60 | !properties plus a keyword | `!propertiesfext:jpg sculpture` | Keyword applied (the advanced form joins them with a space). | search_functions.php:1526, search_functions.php:406 | – | 32,459 | SF9 | must |
| F61 | !properties plus a node | `!propertiesfext:jpg @@389` | Node applied. | do_search.php:111 | – | 41,004 | SF9 | must |
| F62 | !properties, unknown property | `!propertiesbogus:1` | No filter added: everything. | search_functions.php:1539 | – | 100,862 | SF9 | must — unknown property names ignored inside a real property search (bogus:1;fext:jpg = fext:jpg) |
| F63 | !integrityfail | `!integrityfail` | integrity_fail = 1 and a file present; all workflow states. | search_functions.php:1619, search_functions.php:831 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 0 | SF13 | no — a maintenance flag set outside the indexing pipeline; declined |
| F64 | !locked | `!locked` | lock_user <> 0. | search_functions.php:1624 | [Special search terms](https://www.resourcespace.com/knowledge-base/user/special-search-terms) | 1 | SF13 | must |
| F65 | !noningested (admin only) | `!noningested` | For admins: resources with a file_path. For anyone else no branch matches, so a plain search with no terms: everything. | search_functions.php:1629 | – | 100,862 | SF13 | no — F67's bug (an unknown special returns everything); admin half applies no $sql_filter; declined |
| F66 | !report | `!report3p365` | Report 3 over 365 days as results; needs the reports permission, otherwise empty. | search_functions.php:1633, search_functions.php:1663 | – | 0 | SF17 | no |
| F67 | Unknown special search | `!nosuchsearch` | No branch and no plugin hook: a plain search with no terms, so everything. | search_functions.php:1717, do_search.php:382 | – | 100,862 | SF8 | bug — an unknown special search returns everything (was must) |
| F68 | Two special searches | `!last100 !collection3808` | The second is skipped by the keyword stage, and the space makes the !last number invalid (F4): !last1000. | do_search_keywords.php:88 | – | 1,000 | SF8 | bug — Core 7 (was must) |
| F69 | !last, comma before the keyword | `!last100, sculpture` | The comma ends the number, so the last 100 that match (compare F4). | search_functions.php:1127 | – | 100 | SF1 | must — live: refs compared with F1 |
## G. Parameters

do_search() arguments the API exposes: restypes, archive, order_by, sort, fetchrows and offset through do_search; recent_search_daylimit through search_get_previews. The rest (go, editable_only, access, smartsearch, ignore_filters, return_refs_only, return_disk_usage, access_override, returnsql) are reachable only from PHP.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| G1 | One resource type | `sculpture` restypes=1 | resource_type IN (1). | search_functions.php:751 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Simple search](https://www.resourcespace.com/knowledge-base/user/simple-search) | 45,727 | D5 | must |
| G2 | Several resource types | `sculpture` restypes=1,5 | resource_type IN (1,5). | search_functions.php:751 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 48,620 | D5 | must |
| G3 | Resource types "Global" | `sculpture` restypes=Global | Ignored: no type filter (the advanced search "all types" value). | search_functions.php:751 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 | D5 | must |
| G4 | "Global" followed by a type | `sculpture` restypes=Global,5 | Starts with Global, so the whole list is ignored. | search_functions.php:751 | – | 49,468 | D5 | must |
| G5 | Workflow state 0 | `sculpture` archive=0 | archive IN (0). | search_functions.php:851 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 49,468 | D5 | must |
| G6 | Two states | `sculpture` archive=0,8 | archive IN (0,8); a custom state the test users can see. | search_functions.php:851 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 52,848 | D5 | must |
| G7 | Custom state only | `sculpture` archive=8 | archive IN (8). | search_functions.php:851 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 3,380 | D5 | must |
| G8 | State hidden by a z permission | `sculpture` archive=2 | archive IN (2) AND archive NOT IN (1,2,3): nothing for these users. | search_functions.php:873 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 0 | D5 | must — D5's z exclusion includes the uploader override; fixture with an own state-2 resource |
| G9 | Pending submission | `sculpture` archive=-1 | Only the user's own resources in state -1. | search_functions.php:868 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 0 | D5 | must — fixture with own and others' -1 resources |
| G10 | Empty archive | `sculpture` archive= | Default states ($searchstates, else 0). | search_functions.php:839, search_functions.php:1962 | – | 49,468 | D5 | must |
| G11 | Order: relevance (default) | `sculpture` order_by=relevance, fetchrows=0,5 | score, user_rating, hit count, date field, ref. | search_functions.php:3180 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [44274,17206,3962,37462,82074] | D9 | must — accepted difference under Q7.1: keyword searches use the text-match score in place of core's node hit-count score; keyword-less relevance exact |
| G12 | Order: popularity | `sculpture` order_by=popularity, fetchrows=0,5 | user_rating, hit count, date, ref. | search_functions.php:3181 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 49,468 [44274,17206,3962,37462,82074] | SG2 | must — composite key (user_rating, hit_count) plus date and ref; exact as of the last hit-count copy |
| G13 | Order: rating | `sculpture` order_by=rating, fetchrows=0,5 | r.rating, user_rating, score, ref. | search_functions.php:3182 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 49,468 [127577,127561,127494,127493,127491] | SG2 | must — accepted difference for keyword searches: after r.rating and user_rating the text-match score replaces core's hit-count tie-breaker; documented and tested apart from G11 |
| G14 | Order: date | `sculpture` order_by=date, fetchrows=0,5 | $date_field value, then ref. | search_functions.php:3183 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [120365,120367,120373,120363,120372] | D9 | must |
| G15 | Order: colour | `sculpture` order_by=colour, fetchrows=0,5 | has_image, blue, green, red, date, ref. | search_functions.php:3184 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 49,468 [103695,100497,104695,90697,70145] | SG3 | no |
| G16 | Order: title | `sculpture` order_by=title, fetchrows=0,5 | $view_title_field, ref. | search_functions.php:3185 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [113856,113855,113854,113853,113851] | SG1 | must — live collation check |
| G17 | Order: file_path | `sculpture` order_by=file_path, fetchrows=0,5 | file_path, ref. | search_functions.php:3186 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [127577,127561,127494,127493,127491] | SG1 | no — unexercised admin-only key; core answers |
| G18 | Order: resourceid | `sculpture` order_by=resourceid, fetchrows=0,5 | ref. | search_functions.php:3187 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search), [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 49,468 [127577,127561,127494,127493,127491] | D9 | must |
| G19 | Order: resourcetype | `sculpture` order_by=resourcetype, fetchrows=0,5 | resource type order_by, type, ref. | search_functions.php:3188 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [126154,126153,122676,121712,120929] | SG1 | must — index the type's configured order and keep it synchronised; sort by order, type, ref with missing_values first (asc) / last (desc) |
| G20 | Order: extension | `sculpture` order_by=extension, fetchrows=0,5 | file_extension, ref. | search_functions.php:3189 | – | 49,468 [70811,125814,115883,72838,16383] | SG1 | must |
| G21 | Order: status | `sculpture` order_by=status, fetchrows=0,5 | archive, ref. | search_functions.php:3190 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [127577,127561,127494,127493,127491] | SG1 | must — rerun with archive=0,8 |
| G22 | Order: modified | `sculpture` order_by=modified, fetchrows=0,5 | modified, ref. | search_functions.php:3191 | – | 49,468 [108258,108125,108107,106979,106974] | D9 | must |
| G23 | Order: random | `sculpture` order_by=random, fetchrows=0,5 | RAND(). | search_functions.php:3192 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [73411,22273,7893,64079,60516] | SG5 | no |
| G24 | Order: country | `sculpture` order_by=country, fetchrows=0,5 | Only offered when field 3 exists; absent here, so relevance. | search_functions.php:3200, search_functions.php:3249 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [2181,2135,2130,2127,2082] | SG1 | no |
| G25 | Order: a field by number (date) | `sculpture` order_by=field12, fetchrows=0,5 | field12, ref (must be in the resource table joins). | search_functions.php:3216 | [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 49,468 [120365,120367,120373,120363,120372] | D9 | must — field 12 is the date field |
| G26 | Order: numeric field | `numberfield:numrangeneg100\|2000` order_by=field170, sort=asc, fetchrows=0,12 | field170 +0: numeric order, if the field is a resource-table join column. It is not one on the test system: the fallback rewrites the resourceid order with an invalid column, so the query fails and nothing comes back (G43). | search_functions.php:3243 | [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 0 | SG4 | bug — Core 8 (was check); core returns nothing here (Core 8); satisfied by declining |
| G27 | Order: title field by number | `sculpture` order_by=field8, fetchrows=0,5 | field8, ref: text order. | search_functions.php:3245 | [Search order](https://www.resourcespace.com/knowledge-base/resourceadmin/search-order) | 49,468 [113856,113855,113854,113853,113851] | SG4 | must |
| G28 | Order: unknown | `sculpture` order_by=bogus, fetchrows=0,5 | Falls back to relevance. | search_functions.php:3249 | – | 49,468 [44274,17206,3962,37462,82074] | D9 | must |
| G29 | Sort ascending | `sculpture` order_by=resourceid, sort=asc, fetchrows=0,5 | ref ASC. | search_functions.php:3163 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [416,518,1274,1275,1276] | D9 | must |
| G30 | Sort: invalid value | `sculpture` order_by=resourceid, sort=sideways, fetchrows=0,5 | Anything but asc/desc becomes asc. | general_functions.php:5821, do_search.php:95 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [416,518,1274,1275,1276] | D9 | must |
| G31 | Window: offset and count | `sculpture` order_by=resourceid, sort=asc, fetchrows=0,5 | fetchrows "0,5": total plus five rows (structured). | api_bindings.php:20 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 49,468 [416,518,1274,1275,1276] | D10 | must |
| G32 | Window: count only (legacy) | `sculpture` order_by=resourceid, sort=asc, fetchrows=5 | fetchrows "5": five rows, no total. | api_bindings.php:22 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 5 rows [416,518,1274,1275,1276] | D10 | must |
| G33 | Window: legacy with an offset | `sculpture` order_by=resourceid, sort=asc, fetchrows=5, offset=10 | Rows 11 to 15. | api_bindings.php:25 | [do_search API](https://www.resourcespace.com/knowledge-base/api/do_search) | 5 rows [1282,1283,1284,1285,1286] | D10 | must |
| G34 | Recent days: 365 | `sculpture` daylimit=365 | creation_date > today - 365 days (search_get_previews only). | search_functions.php:760, api_bindings.php:65 | [search_get_previews API](https://www.resourcespace.com/knowledge-base/api/search_get_previews) | 10,365 | D5 | must |
| G35 | Recent days: 90 | (empty) daylimit=90 | Everything created in the last 90 days. | search_functions.php:760 | [search_get_previews API](https://www.resourcespace.com/knowledge-base/api/search_get_previews) | 3,089 | D5 | must |
| G36 | Recent days: 30 | (empty) daylimit=30 | Nothing on the test system was created in the last 30 days. | search_functions.php:760 | [search_get_previews API](https://www.resourcespace.com/knowledge-base/api/search_get_previews) | 0 | D5 | must |
| G37 | go (previous / next) | `(any)` | Page direction for view.php; only affects keyword logging in do_search. | do_search.php:44, view.php:57 | – | n/a | D1 | n/a — logging only |
| G38 | editable_only | `(any)` | Adds the edit-permission filter (e states, ert, XE, edit filter). | search_functions.php:928, do_search_filtering.php:62 | – | n/a | D1 | no — deferred under Q6.4 until sync and read-time validation work; declined, core's permission filtering answers; the stale-variable defect is Q11.3's |
| G39 | access | `(any)` | r.access = n, only for users with v. | search_functions.php:920 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | n/a | D6 | must — index the numeric access level; filter only when the user has v and the argument is numeric; frontend fixtures (the API does not expose it) |
| G40 | smartsearch | `(any)` | Keeps the saved archive states instead of the defaults. | search_functions.php:839 | – | n/a | D5 | n/a — via the API; D5 must read $archive_standard when reproducing search_filter() |
| G41 | ignore_filters | `(any)` | Skips date, numrange and fixed-list conversion of field terms. | do_search_keywords.php:128 | – | n/a | D2 | n/a — dead |
| G42.1 | return_refs_only | `(any)` | Returns reference-oriented results rather than normal resource rows; the result shape depends on fetchrows. | do_search.php:44, do_search.php:177 | – | n/a | D1 | must — preserve the fetchrows-dependent shapes; specials omit ORDER BY for refs-only, so compare membership without order |
| G42.2 | return_disk_usage | `(any)` | Wraps the search in SQL that sums disk_usage and counts matching resources. | do_search.php:44, do_search.php:177 | – | n/a | D1 | no — declined to core for its SQL aggregation wrapper |
| G42.3 | returnsql | `(any)` | Returns the prepared SQL query object instead of executing the search. | do_search.php:44, do_search.php:177 | – | n/a | D1 | no — declined so SQL callers receive the query object |
| G42.4 | access_override | `(any)` | Internal override, used by smart collections, that skips specified access and workflow-state restrictions. | do_search.php:44, do_search.php:177 | – | n/a | D1 | must — by construction: override of custom-access joins, T restrictions, confidential filtering, the archive/pending block and z exclusions |
| G43 | Order: a field that is not a join column | `sculpture` order_by=field170, fetchrows=0,5 | check_order_by_in_table_joins() swaps the order for resourceid, but the field branch then writes "resourceid DESC" as a column name: an SQL error, so no results. | search_functions.php:3006, search_functions.php:3245 | – | 0 | SG4 | bug — Core 8 (was must); core returns nothing (Core 8); satisfied by declining |
## H. Combinations

Terms of different kinds in one search. Everything is ANDed: keyword unions, phrase unions, node buckets, date and numeric joins, the special search's own filter, and the parameters.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| H1 | Keyword and node | `sculpture, @@389` | Union join plus node join. | do_search_nodes.php:17 | – | 26,893 | SC1 | must |
| H2 | Keyword and option name | `sculpture, orientation:landscape` | Same as H1. | do_search_keywords.php:253 | – | 26,893 | SC2 | must |
| H3 | Field term and node | `title:sculpture, @@389` | Restricted union plus node join. | do_search_nodes.php:17 | – | 2,439 | SB1 | must |
| H4 | Keyword and two nodes | `sculpture, @@425, @@389` | Two node joins. | do_search_nodes.php:17 | – | 41 | SC1 | must |
| H5 | Field wildcard and node | `title:sculpt*, @@389` | Full-text union plus node join. | do_search_keywords.php:617 | – | 2,507 | SB3 | must |
| H6 | Node and NOT node | `@@425, @@!389` | Join plus NOT EXISTS. | do_search_nodes.php:31 | – | 99 | SC1 | must |
| H7 | Keyword and date | `sculpture, date:2024` | Union plus date join. | do_search_keywords.php:130 | – | 4,517 | SD1 | must |
| H8 | Keyword, node and date, by date | `sculpture, @@389, date:2024` order_by=date, fetchrows=0,5 | All three, ordered by the date field. | search_functions.php:3183 | – | 2,034 [75485,75463,75482,75462,75484] | SD1 | must |
| H9 | Keyword, type, by ref | `sculpture` restypes=5, order_by=resourceid, sort=asc, fetchrows=0,5 | Type filter and ref order. | search_functions.php:751 | – | 2,893 [2181,3879,4093,4094,4096] | D5 | must |
| H10 | Keyword, node, type and two states | `sculpture, @@389` restypes=5, archive=0,2 | State 2 is hidden by z2, so effectively state 0. | search_functions.php:873 | – | 1,408 | D5 | must |
| H11 | Collection and keyword, collection order | `!collection3808 sculpture` order_by=collection, sort=asc, fetchrows=0,5 | Keyword inside the collection, collection order. | search_functions.php:3206 | – | 55 [59367,59354,59348,59320,59315] | SF2 | must |
| H12 | Last 100 and keyword | `!last100 sculpture` fetchrows=0,5 | Newest 100 that match. | search_functions.php:1141 | – | 1,000 [124864,124872,124874,124888,124889] | SF1 | bug — Core 7, the same string as F4: core reads !last1000; Chris's mark: must |
| H13 | Node and NOT keyword | `@@425, -sculpture` | Node join plus NOT IN. | do_search_keywords.php:471 | – | 48 | SA4 | must |
| H14 | Phrase and node | `"sculpture park", @@389` | Phrase union plus node join. | do_search_keywords.php:758 | – | 3,412 | SA3 | must |
| H15 | Keyword and NOT keyword | `sculpture, -landscape` | Union plus NOT IN. | do_search_keywords.php:471 | – | 15,049 | SA4 | must |
| H16 | Keyword and numeric range | `sculpture, numberfield:numrange0\|100` | Union plus numeric join. | do_search_keywords.php:206 | – | 0 | SE1 | must — fixture with a resource holding both |
| H17 | Node and date | `@@389, date:2024` | Node join plus date join. | do_search_keywords.php:130 | – | 4,169 | SD1 | must |
| H18 | OR of options and a node | `materials:bronze;wood, @@389` | OR bucket plus a second bucket. | do_search_nodes.php:17 | – | 18 | SC2 | must |
| H19 | Simple-search year and keyword | `basicyear:2024, sculpture` | Date-field join plus union. | do_search_keywords.php:163 | – | 4,517 | SD2 | must |
| H20 | Simple-search year and node | `basicyear:2024, @@389` | Date-field join plus node join. | do_search_keywords.php:163 | – | 4,169 | SD2 | must |
| H21 | Two date fields | `date:2024, expirydate:2029` | Two date joins. | do_search_keywords.php:130 | – | 2,972 | SD1 | must |
| H22 | Two wildcards | `sculpt*, land*` | Two full-text unions. | do_search_keywords.php:617 | – | 36,026 | SA5 | must — each term a prefix over every eligible visible field; tokenisation parity and candidate caps (max_filter_by_candidates) proved by SA5's battery |
| H23 | Phrase and NOT phrase | `"sculpture park", -"black and white"` | Phrase union plus phrase NOT IN. | do_search_keywords.php:828 | – | 6,758 | SA4 | no |
| H24 | Four keywords | `sculpture landscape nature outdoor` | Four unions ANDed. | do_search_union_assembly.php:55 | – | 19,608 | SA1 | must |
| H25 | Two fields, node and date | `title:sculpture, credit:wilde, @@389, date:2024` | Two restricted unions, node join, date join. | do_search_union_assembly.php:38 | – | 65 | SB1 | must |
| H26 | Collection and date | `!collection3808 date:2025` | Date join inside the collection. | search_functions.php:1285 | – | 55 | SF2 | must |
| H27 | Contributions, keyword and node | `!contributions9 sculpture, @@389` | created_by filter plus union plus node. | search_functions.php:1457 | – | 9,240 | SF5 | must |
| H28 | List and node | `!list416:4460:8692 @@389` | Ref list plus node join. | search_functions.php:1480 | – | 1 | SF3 | must |
| H29 | Hasdata and node | `!hasdata170 @@389` | Field join plus node join. | search_functions.php:1512 | – | 5 | SF6 | must |
| H30 | Properties and date | `!propertiesfext:jpg date:2024` | Extension filter plus date join. | search_functions.php:1524 | – | 6,224 | SF9 | must |
| H31 | Everything at once | `sculpture, "sculpture park", @@389, orientation:landscape, date:2024, -"black and white"` restypes=1, archive=0, order_by=date, sort=desc, fetchrows=0,5 | Keyword, phrase, node, option (same node twice), date, NOT phrase, type, state, order. | do_search.php:412 | – | 307 [47432,47431,47430,47429,47428] | SH1 | must — the fixture proves the NOT phrase and the adjacency separately; expectation under "H31, the capstone" in checks-findings.md |
| H32 | Bare OR and a node | `sculpture;landscape, @@389` | Ran after B5 had created the keyword "sculpture;landscape", so the OR worked: sculpture or landscape, and every resource with that orientation node also carries the word landscape, hence the same total as @@389 alone. Before B5 it would be a suggestion (A16). | do_search_keywords.php:387 | – | 58,028 | SA7 | bug — Core 9; Chris's mark: check |
| H33 | Wildcard OR and a node | `sculpt*;land*, @@389` | RLIKE union plus node join. | do_search_keywords.php:648 | – | 58,028 | SA7 | no |
| H34 | Field OR and a node | `title:sculpture;park, @@389` | Alternative keywords in the field plus node join. | do_search_keywords.php:301 | – | 2,653 | SA7 | must |
| H35 | Date range, node and keyword | `date:rangestart2024-01-01end2024-12-31, @@389, sculpture` | Range join, node join, union. | do_search_keywords.php:180 | – | 2,030 | SD3 | must |
| H36 | Year as a word and a keyword | `2024, sculpture` | Two keyword unions (2024 is a keyword from date indexing). | search_functions.php:2059 | – | 5,477 | SA9 | must |
| H37 | Type name and keyword | `document sculpture` | Type union plus keyword union. | do_search_keywords.php:56 | – | 111 | SA9 | must |
| H38 | Resource types and collection ignored together | `!collection3618 sculpture` restypes=5, archive=8 | Types and states are both ignored for a collection search; the keyword still applies. | search_functions.php:751, search_functions.php:825 | – | 409 | SF2 | must — the parameters are ignored, z still applies |
## I. Syntax edge cases

Strings a user might type that core accepts but reads differently from what was meant.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| I1 | No space after a special search | `!collection3808,sculpture` | Keywords are only taken after the first space: the keyword is dropped. | do_search.php:126 | – | 55 | D2 | no |
| I2 | Comma without a space | `sculpture,landscape` | A string with no whitespace is kept as one keyword; the comma then counts as punctuation, so core searches the phrase "sculpture landscape", not two words. | search_functions.php:2157 | – | 22 | SA6 | bug — Core 10; Chris's mark: check |
| I3 | Comma inside a field term | `title:sculpture,landscape` | Quoted because of the comma, then split: the phrase "sculpture landscape" in title. | do_search_keywords.php:286 | – | 0 | SB4 | bug — Core 10, the field form of I2; was check; same disposition as I2 |
| I4 | Repeated word | `sculpture sculpture` | Duplicates removed: same as A1. | do_search.php:146 | – | 49,468 | D2 | must |
| I5 | Surrounding spaces | `   sculpture   ` | Trimmed. | do_search.php:145 | – | 49,468 | D2 | must |
| I6 | Trailing colon | `sculpture:` | Field "sculpture" does not exist: colon becomes a space, so the word sculpture. | do_search_keywords.php:112 | – | 49,468 | SB1 | must — deterministic |
| I7 | Leading colon | `:sculpture` | Empty field name: same as I6. | do_search_keywords.php:112 | – | 49,468 | SB1 | must — the plugin's field lookup is literal like core's |
| I8 | Unbalanced quote | `"sculpture` | Not a quoted string (no closing quote); the quote is a separator. | do_search_keywords.php:21 | – | 49,468 | SA3 | no — an accidental path; declined |
| I9 | Upper-case special search | `!LAST100` | Prefixes are compared case-sensitively: not !last, so an unknown special search (F67). | search_functions.php:1110 | – | 100,862 | SF8 | bug — an unknown special search returns everything (F67); Chris's mark: check |
| I10 | Nodes without a space | `@@389,@@425` | Both tokens found; the comma is left behind and ignored. | do_search.php:529 | – | 49 | SC1 | must |
| I11 | Upper-case field name and value | `Title:Sculpture` | Field names compare case-insensitively in MySQL; the value is normalised. | do_search_keywords.php:105 | – | 4,226 | SB1 | must |
| I12 | Dropdown value from the advanced form | `"source:igital Camer"` | search_form_to_search_query has a branch that strips the first and last characters of a dropdown value with spaces; dropdowns post node refs instead, so the branch is unreachable and this is only what it would send for "Digital Camera". | search_functions.php:290 | – | 0 | SJ1 | no — unreachable branch |
## J. Strings as the advanced search page assembles them

search_form_to_search_query() builds one string in a fixed order: the date dropdowns, the words of the all-fields box, a !list prefix for the resource IDs box, the full-text box, one term per word per field box (a quoted phrase stays quoted around the whole term), the ticked options and dropdown choices as @@ tokens, and last a !properties prefix joined with " ,". A posted special search such as !contributions is prefixed with a comma and no space. Not sent as text: a checkbox list with every option ticked (omitted), the Featured Collections type (a collections search) and the access selector (a PHP argument only). Fields with a verbatim-keyword regex keep their value unsplit; none is configured on the test system.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| J1 | Resource IDs, words, a field, nodes, type and state | `!list2181:3879:4093 sculpture, landscape, title:sculpture, @@389@@388` restypes=5, archive=0 | The !list prefix, then after a space the terms the form added: the listed refs that also match everything else. | search_functions.php:239, do_search.php:126 | – | 1 | SJ1 | must |
| J2 | The same with a property filled in | `!propertiesfext:jpg ,!list2181:3879:4093 sculpture, landscape, title:sculpture, @@389@@388` restypes=5, archive=0 | The !properties prefix is now the special search and !list a second one, which the keyword stage skips: the resource IDs box is ignored whenever a property is set. | search_functions.php:406, do_search_keywords.php:88 | – | 8 | SJ1 | bug — the !properties and !list prefixes collide; Chris's mark: must |
| J3 | Posted special search prefix | `!contributions9,sculpture` | The form joins them with a comma and no space, so the term is dropped (I1): the contributions alone. | search_functions.php:410, do_search.php:126 | – | 41,283 | SJ1 | must |
| J4 | Posted special search prefix, more terms | `!contributions9,title:park, sculpture` | Only what follows the first space is kept: sculpture applies, title:park is lost. | do_search.php:126 | – | 19,396 | SJ1 | must — the ref parse takes the leading integer |
| J5 | Range with a start year only | `date:rangestart2024` | The form sends the year unpadded: name >= "2024", so everything from 2024 on. | search_functions.php:3348, do_search_keywords.php:193 | – | 21,151 | SD3 | must |
| J6 | Range with months only | `date:rangestart2024-05end2024-06-31` | Start month unpadded, end month padded with 31: May and June 2024. | search_functions.php:3348 | – | 1,947 | SD3 | must |
| J7 | Live result count | `sculpture` fetchrows=0,0 | The count the page shows while the form is filled in: fetchrows 0,0 gives the total and no rows. | search_advanced.php:82, api_bindings.php:20 | [Advanced search](https://www.resourcespace.com/knowledge-base/user/advanced-search) | 49,468 | D10 | must |
| J8 | Checkbox list, two options ticked | `sculpture, @@6125@@6130` | Ticked options are sent as one OR token per field. | search_functions.php:364 | – | 284 | SC2 | must |
| J9 | Checkbox list with $checkbox_and | `sculpture, @@6125, @@6130` | Each ticked option becomes its own token: AND. Both options resolve here, unlike the name form in C7. | search_functions.php:384 | – | 0 | SC2 | must — live: each node alone > 0 |
| J10 | Dropdown choice | `sculpture, @@247` | A dropdown field sends one node token. | search_functions.php:374 | – | 1,429 | SC2 | must |
| J11 | Property filter with terms | `!propertiesfext:jpg ,sculpture, title:park` | The form's separator is a space then a comma; the comma is dropped and the terms apply. | search_functions.php:406 | – | 547 | SJ1 | must |
| J12 | Full-text box with a word | `sculpture, "@FULL_TEXT:distant island"` | The full-text term is appended with a space and then normalised into the comma list. | search_functions.php:242 | – | 461 | SA8 | must |
| J13 | Date value and range on one field | `date:2024-05, date:rangestart2024-05-01end2024-05-31` | The date boxes and the range boxes are separate inputs, so both terms are sent: two joins, ANDed. | search_functions.php:3289 | – | 524 | SD3 | must — three joins ANDed; fixture with a 2024-05-00 value |
## K. What the indexing flags control

Only the values of fields flagged for indexing become keywords, so the flag governs every form that goes through the keyword table: plain words, field words, phrases, OR, NOT and the short-wildcard LIKE branch. It does not govern full-text and other wildcards, which read node names, nor date and number terms, fixed-list options by name, @@ tokens, !hasdata and !empty, which all read nodes. Partial indexing adds every fragment of three or more letters as a keyword at the word's position. Complete indexing stores the whole value as one keyword, never split, so a value with spaces cannot be typed as a word. An inactive field, or one hidden from the user, is dropped from every union, wildcards included. Only the first 500 characters of a value become keywords ($node_keyword_index_chars). The advanced search form lists indexed fields only; the simple bar needs indexed or fixed-list. See also B12, B13, B19, C8, C9, D22, D23, F43 and B14.

| # | Search | String and parameters | What core does | Where | Documented | Core | Package | Required |
|---|---|---|---|---|---|---|---|---|
| K1 | Word that exists only in a non-indexed field | `nikon` | Not a keyword: nothing (the value is in the camera field, not flagged for indexing). | do_search_keywords.php:387, resource_functions.php:7840 | [Indexing metadata](https://www.resourcespace.com/knowledge-base/resourceadmin/indexing-metadata), [Field configuration](https://www.resourcespace.com/knowledge-base/resourceadmin/basic-configure-metadata-field) | 0 | SK1 | must |
| K2 | The same word with a wildcard | `nikon*` | The full-text branch reads node names of every visible field, indexed or not: the camera values match. | do_search_keywords.php:617 | – | 23,412 | SK1 | must — fixture must not assert the suggestion text after a wildcard run |
| K3 | Wildcard inside the non-indexed field | `camera:nik*` | Full-text restricted to the field: matches, although camera:nikon (B12) does not. | do_search_keywords.php:617 | – | 23,412 | SK1 | must — fixture: nik* matches, ni* and nikon do not |
| K4 | Full-text box on a non-indexed field | `"@FULL_TEXT:nikon"` | MATCH over node names: matches. | do_search_keywords.php:30 | – | 23,412 | SA8 | must |
| K5 | Bare fragment of a partially indexed field | `ksho` | Fragments are ordinary keywords, so the fragment matches outside the field term too (compare B14). | search_functions.php:2270 | [Indexing metadata](https://www.resourcespace.com/knowledge-base/resourceadmin/indexing-metadata) | 665 | SB6 | must |
| K6 | Word in an inactive field | `description:sculpture` | The field is found by name but inactive fields count as hidden and are excluded from the union: nothing (E7 in the review). | resource_functions.php:7871, do_search_keywords.php:511 | – | 0 | SK2 | must |
| K7 | Phrase held only in non-indexed fields | `"library book"` | The phrase goes through keywords: nothing. | do_search_keywords.php:758 | – | 0 | SK1 | must |
| K8 | The same phrase through full text | `"@FULL_TEXT:[QUOTES]library book[QUOTES]"` | Node names are read: the active non-indexed field matches, the inactive one is excluded as hidden. | do_search_keywords.php:30, do_search_keywords.php:40 | – | 1 | SA8 | must — fixture with the phrase on two resources |
| K9 | Complete indexing | `field:two words` | The whole value is one keyword; a typed search splits on spaces, so only a single-token value is reachable by word, and wildcards or full text otherwise. No complete-indexed field on the test system. | node_functions.php:1082 | – | n/a | SK3 | must — whole-value keyword (_c, SK3 schema addition); no such field on the test system |
| K10 | Word beyond the 500th character | `field:word` | Only the first $node_keyword_index_chars characters become keywords; a wildcard still reads the whole value (E10 in the review). The one indexed value over 500 characters on the test system is unused markup, so not measured. | node_functions.php:1077 | [Indexing metadata](https://www.resourcespace.com/knowledge-base/resourceadmin/indexing-metadata), [Full text searching](https://www.resourcespace.com/knowledge-base/user/fulltext-search) | n/a | SK3 | must — fixture: unique word at character 600: word 0, wor* 1, *ord 0, full-text box 1 (uncapped _w, SK3 schema addition) |
| K11 | Number range on a non-indexed numeric field | `field:numrange1\|10` | Node join, so it works regardless of the flag, as dates do (D22). No such field on the test system. | do_search_keywords.php:206 | – | n/a | SE1 | must — fixture only |
| K12 | Field hidden from the user by permission | `field:word` | A field term on it returns false before the plugin is asked; its values are excluded from every union. Both test users can see every field, so not measured. | do_search_keywords.php:122, resource_functions.php:7871 | – | n/a | SK2 | must — core returns false before the hook |

