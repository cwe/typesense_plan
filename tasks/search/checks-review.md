# Check rows for manual review

Review entries for the 77 catalogue rows marked *check* (generated 2026-10-27 from `catalogue.md` and
[checks-findings.md](checks-findings.md)). **Claim** is what the catalogue says core does; **Held?** is
what reading core's code found (the citations are in the findings file); **Proposed** is the mark
suggested there and what, if anything, still needs a run. Write the final mark in **Decision**
(`must`, `no`, `n/a`, `bug`) and it is copied into the catalogue; a blank row stays *check*.

G42 is split into four subentries, giving 80 review entries.

Proposed marks: bug 3 · must 61 · n/a 6 · no 10.

Decided (Chris, 2026-10-29): bug 2 · must 61 · n/a 3 · no 14. Applied to `catalogue.md` and `coverage.md` on
2026-10-30; the catalogue descriptions of A12, A13, B7, C5, D25 and B17 were corrected at the same time.


## Group A

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| A10 | NOT a phrase: `-"sculpture park"` | SA4 | Everything except resources with the phrase. | yes | **must**; fixture "park sculpture" not excluded | must |
| A12 | Leading wildcard: `*scape` | SA5 | RLIKE on node names (".*?scape" at a word boundary). | **No** — not RLIKE over node names | **must**, rule corrected: keyword-suffix LIKE over indexed fields (fixture: value only in a non-indexed field → 0; in a partial-indexed field → infix match) | must |
| A13 | Wildcard inside a word: `sc*pture` | SA5 | Goes to the full-text branch as "+sc*pture"; MySQL honours only the trailing part of the pattern, so this behaves like sc*. | partly | **no** — a tokenizer accident; decline (optional live check: `sc*zzzz` = `sc*pture`) | no |
| A15 | OR of two wildcards: `land*;sun*` | SA7 | Semicolon forces the RLIKE branch: "(land.*?\|sun.*?)". | yes | **must** for the single-token / `field:` form; fixture must show the whitespace collapse | must |
| A17 | Stop word alone: `the` | SA2 | Skipped ($noadd); no terms left, so everything. | yes | **must**, with the case rule copied exactly (live check `The`) | must |
| A30 | Misspelt word: `sculptre` | SA11 | An unknown word gives no results and a sound-alike suggestion instead of an array; on the test system this misspelling happens to exist as a keyword, so A31 shows the real case. | mechanism yes; string no | **n/a** (core answers first); replace the string with one that cannot be a keyword and run it before any B/I row uses the word | no |

## Group B

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| B3 | Quoted phrase in a field: `"title:henry moore"` | SB2 | Phrase search restricted to the field (the form the advanced search sends). | yes | **must**; fixture "Moore, Henry" (B2 matches, B3 must not) | must |
| B4 | Quotes after the colon: `title:"henry moore"` | SB2 | Not the supported form: the tokens are title:"henry and moore", but the stray quotes are stripped, so it works out the same as B2. | partly — "quotes stripped" is really re-quote plus swallow | **no** — accidental parse; decline | no |
| B7 | Leading wildcard in a field: `title:*scape` | SB3 | RLIKE restricted to the field. | **No** — LIKE on keywords, not RLIKE | **must**, corrected; live `camera:*ikon` = 0 proves LIKE | must |
| B17 | NOT after the colon: `title:-sculpture` | SB5 | Quoted, then the word "-sculpture" is omitted: excludes nothing. | partly — not "omitted": a NOT against a junk keyword | **bug** — NOT inside a field term silently ignored and pollutes `keyword`; decline meanwhile | bug |

## Group C

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| C4 | One word of an option: `usagerights:social` | SC3 | Not an option name, so a keyword search restricted to the field. | yes | **must**; fixture with two options sharing a word | must |
| C5 | Wildcard on an option: `usagerights:soc*` | SC3 | LIKE on keywords within the field. | **No** — mechanism wrong | **must**, rule corrected; fixture: 2-letter wildcard, non-indexed fixed-list field | must |
| C7 | AND of options: `materials:bronze, materials:wood` | SC3 | Two buckets: resources with both (the $checkbox_and form). On the test system the first option's name is stored with a leading byte-order mark, so it is not matched as an option and becomes a keyword search within the... | yes, BOM aside | **must**; fixture with clean names and resources holding one, both, neither | must |
| C12 | One word of a dynamic keyword: `artistname:moore` | SC3 | Keyword search in the field. | yes | **must**; fixture with two options sharing a word | must |
| C23 | Option name and a node of the same field: `orientation:landscape, @@388` | SC2 | Two buckets on one field: both options needed. | yes | **must**; fixture confirming 388's field | must |

## Group D

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| D4 | Date stored with a time: `date:2017-05-10` | SD1 | The prefix still matches a value stored with a time of day. | yes | **must**; fixture with a timed value | must |
| D5 | Range: `date:rangestart2024-01-01end2024-02-29` | SD3 | name >= start AND name <= "end 23:59:59". | yes | **must** (Q3.4 (a) reproduces it); fixture with a `-00` partial and a timed end-day value | must |
| D7 | Range, end only: `date:rangeend2005-12-31` | SD3 | name <= end. | yes | **must**; add a row for the stale-bound quirk (`date:rangestart…, date:rangeend…`) | must |
| D8 | Range as the EDTF box sends it: `date:rangestart2024-00-00end2024-12-99` | SD3 | "2024/2024" in the EDTF box becomes this; string comparison still brackets the year. | yes | **must**; fixture with a `2024-12-31 23:xx` value | must |
| D9 | Range, end month without a day: `date:rangestart2024-01-01end2024-02-31` | SD3 | The form pads a missing day with 31; as a string "2024-02-31" is after every February date. | yes | **must** | must |
| D16 | startdate / enddate form inputs: `startdate:2024-01-01, enddate:2024-12-31` | SD2 | search_form_to_search_query still assembles these, but no page posts the inputs and nothing in do_search handles them: unknown field names, so words and a suggestion. | yes | **n/a** | no |
| D22 | Date field not flagged for indexing: `datephototaken:2024` | SD5 | Date terms join the node table directly, so the index flag does not matter. | yes | **must**; fixture only | must |
| D23 | Same field, full date: `datephototaken:2012-01-26` | SD5 | Matches values stored with a time of day. | yes | **must** | must |
| D25 | Date-range field, range: `daterangetest:rangestart2020-01-01end2021-12-31` | SD4 | Start value >= start AND end value <= end, as strings. | **No** | **must** with the rule corrected to overlap (two `_k` values per resource; `_k:>=S && _k:<=E` over an array gives exactly this); fixture: partial overlap, enclosing, outside, open-ended | must |
| D27 | OR on a date: `date:2024;2025` | SD1 | No OR for dates: LIKE "2024;2025%", nothing. | yes | **no** — no form emits it; declining gives core's empty answer | no |

## Group E

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| E6 | Plain number in the field: `numberfield:42` | SE1 | Not a range: an ordinary keyword search for "42" in the field. | yes | **must** (it is SB1); fixture with `42.5`, `042` | must |
| E7 | numrange on a text field: `title:numrange1\|10` | SE1 | Accepted for any type 0 field: compares title values as numbers. | yes | **no** — the form only emits numrange for `field_constraint = 1` fields (`render_functions.php:540-578`); incidental casting; decline | no |

## Group F

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| F37 | !unused: `!unused` | SF16 | In no collection. | yes | **must** if memberships are indexed (they are, with every collection type — Q5.7 decides which kinds). **Note (Chris, 2026-10-29):** core's query is `r.ref NOT IN (SELECT c.resource FROM collection_resource c)` with no join on `collection.type` (`search_functions.php:1471-1484`), so a resource in only a user's private (type 0), selection (2), upload (1), share-upload (5) or request (6) collection is "used". Chris is not sure core *should* count the private and selection kinds; it is a core question (the documentation says "in no collection"). The membership index excludes selection, upload and share-upload collections (P27), so serving F37 exactly either widens what is indexed (Q5.7) or waits for core to decide what `!unused` means — see Q5.7 and SF16. | must |
| F39 | !listall: `!listall416:4460:8692` | SF3 | Same, by name "all"; the filter difference is in search_filter, which treats both alike. | yes | **must** — treat as `!list` | must |
| F48 | !empty with a resource type: `!empty170` restypes=1 | SF10 | Candidates limited to the given types. | partly — right outcome, wrong mechanism | **must** (type filter by construction); DB: is field 170 global? | must |
| F52 | !properties, extension wildcard: `!propertiesfext:jp*` | SF9 | * becomes %: jpg and jpeg. | yes for the mechanism | **must**, expectation written as the LIKE | must |
| F59 | !properties orientation: `!propertiesorientation:portrait` | SF9 | Compares resource_dimensions height and width. | yes | **must**; fixture with a dimension-less resource | must |
| F62 | !properties, unknown property: `!propertiesbogus:1` | SF9 | No filter added: everything. | yes | **must** — unknown property names ignored inside a real property search (fixture: `bogus:1;fext:jpg` = `fext:jpg`) | must |
| F63 | !integrityfail: `!integrityfail` | SF13 | integrity_fail = 1 and a file present; all workflow states. | partly | **no** — maintenance flags set outside the indexing pipeline; decline | no |
| F65 | !noningested (admin only): `!noningested` | SF13 | For admins: resources with a file_path. For anyone else no branch matches, so a plain search with no terms: everything. | yes, plus the unfiltered admin half | **bug** — F67's; decline meanwhile | no |
| F69 | !last, comma before the keyword: `!last100, sculpture` | SF1 | The comma ends the number, so the last 100 that match (compare F4). | yes | **must**; live: refs compared with F1, or a rare word | must |

## Group G

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| G8 | State hidden by a z permission: `sculpture` archive=2 | D5 | archive IN (2) AND archive NOT IN (1,2,3): nothing for these users. | partly: "nothing" only if the override is off | **must** — D5's z exclusion must include the uploader override; fixture with an own state-2 resource | must |
| G9 | Pending submission: `sculpture` archive=-1 | D5 | Only the user's own resources in state -1. | yes | **must**; fixture with own and others' -1 resources | must |
| G11 | Order: relevance (default): `sculpture` order_by=relevance, fetchrows=0,5 | D9 | score, user_rating, hit count, date field, ref. | yes | **must**; fixture with a non-zero node hit count on a matched node. Accepted difference under [Q7.1](../../decisions.md): keyword searches use Typesense text-match score in place of core's matched-node hit-count score, followed by the shared ordering keys; relevance order may therefore differ and must be documented. Keyword-less relevance ordering remains exact. | must |
| G12 | Order: popularity: `sculpture` order_by=popularity, fetchrows=0,5 | SG2 | user_rating, hit count, date, ref. | yes | **must**; fixture with rated resources | must |
| G13 | Order: rating: `sculpture` order_by=rating, fetchrows=0,5 | SG2 | r.rating, user_rating, score, ref. | yes for the SQL; the note was incomplete (ref order follows only because `score` also ties) | **must**; accepted difference for keyword searches: after `r.rating` and `user_rating`, Typesense text-match score replaces core's matched-node hit-count score as the tie-breaker, followed by ref. Rating order may therefore differ when both rating keys tie; document and test this separately from G11's relevance-order exception. | must |
| G16 | Order: title: `sculpture` order_by=title, fetchrows=0,5 | SG1 | $view_title_field, ref. | yes | **must**; live collation check | must |
| G17 | Order: file_path: `sculpture` order_by=file_path, fetchrows=0,5 | SG1 | file_path, ref. | yes | **no** — unexercised, admin-only key; core answers | no |
| G19 | Order: resourcetype: `sculpture` order_by=resourcetype, fetchrows=0,5 | SG1 | resource type order_by, type, ref. | yes | **must**; index the type's configured order and keep it synchronised. Sort by type order, type ID, then ref. Preserve MySQL NULL ordering with Typesense `missing_values:first` for ascending and `missing_values:last` for descending, e.g. `resource_type_order(missing_values:first):asc,resource_type:asc,ref:asc`. See [Typesense 30.2 missing-value sorting](https://typesense.org/docs/30.2/api/search.html#sorting-null-empty-or-missing-values). | must |
| G20 | Order: extension: `sculpture` order_by=extension, fetchrows=0,5 | SG1 | file_extension, ref. | yes | **must** | must |
| G21 | Order: status: `sculpture` order_by=status, fetchrows=0,5 | SG1 | archive, ref. | yes | **must**; rerun with `archive=0,8` | must |
| G25 | Order: a field by number (date): `sculpture` order_by=field12, fetchrows=0,5 | D9 | field12, ref (must be in the resource table joins). | yes | **must**; DB: field 12's `sort_method`, `field_constraint` | must |
| G27 | Order: title field by number: `sculpture` order_by=field8, fetchrows=0,5 | SG4 | field8, ref: text order. | yes | **must** | must |
| G37 | go (previous / next): `(any)` | D1 | Page direction for view.php; only affects keyword logging in do_search. | yes | **n/a** | n/a |
| G38 | editable_only: `(any)` | D1 | Adds the edit-permission filter (e states, ert, XE, edit filter). | yes, plus omitted clauses | **no**, deferred under [Q6.4](../../chunks/06-access-permissions-and-validation.md#q64--editable-only-searches) until synchronisation and read-time validation are working and the complete edit-permission rules, custom grants and edit hooks can be supported. Decline editable-only searches to core meanwhile so editing workflows continue to use core's permission filtering. The stale-variable defect in the custom-state edit loop is a separate core issue for Q11.3. | no |
| G39 | access: `(any)` | D6 | r.access = n, only for users with v. | yes | **must** for frontend searches; index and synchronise the stored numeric access level. In PHP, require `v` and a non-null numeric access argument before adding the corresponding Typesense equality filter (e.g. `access:1`); otherwise omit this optional filter as core does. Preserve all other access restrictions. Test users with and without `v`. The standard search API does not expose this argument, so use frontend or direct-core fixtures. | must |
| G40 | smartsearch: `(any)` | D5 | Keeps the saved archive states instead of the defaults. | yes, with that context | **n/a** via API; **D5 must read `$archive_standard`** when reproducing `search_filter()` | n/a |
| G41 | ignore_filters: `(any)` | D2 | Skips date, numrange and fixed-list conversion of field terms. | yes — dead | **n/a** | n/a |
| G42.1 | return_refs_only: `(any)` | D1 | Returns reference-oriented results rather than normal resource rows; the result shape depends on fetchrows. | yes | **must**; preserve fetchrows-dependent shapes. Special searches omit ORDER BY for refs-only results, so compare their membership without requiring order. | must |
| G42.2 | return_disk_usage: `(any)` | D1 | Wraps the search in SQL that sums disk_usage and counts matching resources. | yes | **no** — decline to core for its SQL aggregation wrapper. | no |
| G42.3 | returnsql: `(any)` | D1 | Returns the prepared SQL query object instead of executing the search. | yes | **no** — decline to core so SQL callers receive the expected query object. | no |
| G42.4 | access_override: `(any)` | D1 | Internal override, used by smart collections, that skips specified access and workflow-state restrictions. | yes | **must** by construction; reproduce the override of custom-access joins, T restrictions, confidential-resource filtering, the normal archive/pending block and z exclusions. Preserve checks outside those override gates. | must |

## Group H

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| H11 | Collection and keyword, collection order: `!collection3808 sculpture` order_by=collection, sort=asc, fetchrows=0,5 | SF2 | Keyword inside the collection, collection order. | yes | **must**; DB: `sortorder`, `date_added` of 3808's members | must |
| H16 | Keyword and numeric range: `sculpture, numberfield:numrange0\|100` | SE1 | Union plus numeric join. | yes | **must**; fixture with a resource holding both | must |
| H22 | Two wildcards: `sculpt*, land*` | SA5 | Two full-text unions. | yes | **must**; Typesense `prefix=true` applies only to the last query word. Prefix filters combined with AND are a possible implementation; each term must search all eligible visible fields, including non-keyword-indexed fields, and may match a different field. Verify tokenisation parity and prefix-expansion completeness (`max_filter_by_candidates` defaults to 4); implementation parity remains unproved. See [Typesense 30.2 filter parameters](https://typesense.org/docs/30.2/api/search.html#filter-parameters). | must |
| H28 | List and node: `!list416:4460:8692 @@389` | SF3 | Ref list plus node join. | yes | **must**; DB: node 389 on the three refs | must |
| H29 | Hasdata and node: `!hasdata170 @@389` | SF6 | Field join plus node join. | yes | **must** | must |
| H31 | Everything at once: `sculpture, "sculpture park", @@389, orientation:landscape, date:2024, -"black and white"` restypes=1, archive=0, order_by=date, sort=desc, fetchrows=0,5 | SH1 | Keyword, phrase, node, option (same node twice), date, NOT phrase, type, state, order. | yes | **must**; the fixture must prove the NOT phrase and the adjacency separately; the full expectation is written out under "H31, the capstone" in checks-findings.md | must |
| H38 | Resource types and collection ignored together: `!collection3618 sculpture` restypes=5, archive=8 | SF2 | Types and states are both ignored for a collection search; the keyword still applies. | partly — "states ignored" means the parameter | **must**; DB: 3618's members by type and state | must |

## Group I

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| I3 | Comma inside a field term: `title:sculpture,landscape` | SB4 | Quoted because of the comma, then split: the phrase "sculpture landscape" in title. | yes | **bug** — the field form of I2 (Core 10); same disposition as I2 | bug |
| I6 | Trailing colon: `sculpture:` | SB1 | Field "sculpture" does not exist: colon becomes a space, so the word sculpture. | yes | **must** — deterministic | must |
| I7 | Leading colon: `:sculpture` | SB1 | Empty field name: same as I6. | yes on the test system | **must**, provided the plugin's field lookup is literal like core's | must |
| I8 | Unbalanced quote: `"sculpture` | SA3 | Not a quoted string (no closing quote); the quote is a separator. | partly | **no** — accidental path; decline | no |
| I12 | Dropdown value from the advanced form: `"source:igital Camer"` | SJ1 | search_form_to_search_query has a branch that strips the first and last characters of a dropdown value with spaces; dropdowns post node refs instead, so the branch is unreachable and this is only what it would send fo... | yes | **n/a** | no |

## Group J

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| J4 | Posted special search prefix, more terms: `!contributions9,title:park, sculpture` | SJ1 | Only what follows the first space is kept: sculpture applies, title:park is lost. | yes, plus the coercion | **must**; the plugin's ref parse takes the leading integer | must |
| J9 | Checkbox list with $checkbox_and: `sculpture, @@6125, @@6130` | SC2 | Each ticked option becomes its own token: AND. Both options resolve here, unlike the name form in C7. | yes | **must**; live: each node alone > 0 | must |
| J13 | Date value and range on one field: `date:2024-05, date:rangestart2024-05-01end2024-05-31` | SD3 | The date boxes and the range boxes are separate inputs, so both terms are sent: two joins, ANDed. | yes (three joins, not two) | **must**; fixture with a `2024-05-00` value | must |

## Group K

| Row | Search | Package | Catalogue claim | Held? | Proposed mark and what remains | Decision |
|---|---|---|---|---|---|---|
| K2 | The same word with a wildcard: `nikon*` | SK1 | The full-text branch reads node names of every visible field, indexed or not: the camera values match. | yes | **must**; fixture must not assert the suggestion text after a wildcard run | must |
| K3 | Wildcard inside the non-indexed field: `camera:nik*` | SK1 | Full-text restricted to the field: matches, although camera:nikon (B12) does not. | yes | **must**; fixture: `nik*` matches, `ni*` and `nikon` do not | must |
| K4 | Full-text box on a non-indexed field: `"@FULL_TEXT:nikon"` | SA8 | MATCH over node names: matches. | yes | **must** | must |
| K8 | The same phrase through full text: `"@FULL_TEXT:[QUOTES]library book[QUOTES]"` | SA8 | Node names are read: the active non-indexed field matches, the inactive one is excluded as hidden. | yes | **must**; fixture with the phrase on two resources | must |
| K9 | Complete indexing: `field:two words` | SK3 | The whole value is one keyword; a typed search splits on spaces, so only a single-token value is reachable by word, and wildcards or full text otherwise. No complete-indexed field on the test system. | partly | **must** — whole-value keyword; fixture as listed | must |
| K10 | Word beyond the 500th character: `field:word` | SK3 | Only the first $node_keyword_index_chars characters become keywords; a wildcard still reads the whole value (E10 in the review). The one indexed value over 500 characters on the test system is unused markup, so not me... | partly: "a wildcard reads the whole value" only for full text and RLIKE; `*ord`, `wo*` and stop-word-prefixed wildcards are capped | **must** — fixture: unique word at character 600: `word` 0, `wor*` 1, `*ord` 0, full-text box 1 | must |
| K11 | Number range on a non-indexed numeric field: `field:numrange1\|10` | SE1 | Node join, so it works regardless of the flag, as dates do (D22). No such field on the test system. | yes | **must**; fixture only | must |
| K12 | Field hidden from the user by permission: `field:word` | SK2 | A field term on it returns false before the plugin is asked; its values are excluded from every union. Both test users can see every field, so not measured. | yes | **must** | must |

