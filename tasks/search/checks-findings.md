# Check findings — core read on 2026-10-22

What core's code actually does for the check rows that can be settled by reading it, with the
proposed mark for each. Four read-only passes over `include/do_search.php` (`ds`),
`include/do_search_keywords.php` (`dsk`), `include/do_search_nodes.php` (`dsn`),
`include/do_search_union_assembly.php` (`dsu`), `include/search_functions.php` (`sf`),
`include/node_functions.php` (`nf`), `include/resource_functions.php` (`rf`) at the working tree of
`cwe/resourcespace` (search code identical to upstream; `rf` lines above 7839 are 32 higher than
upstream's because of the fork's helper). **Marks are proposals until Chris sets them.** "Wrong
reason" = the live total could match the catalogue's description without the described mechanism
being the cause. Config defaults assumed: `$stemming` off, `$uploader_view_override` on,
`$user_rating` off, `$orderbyrating` off, `$collections_omit_archived` off.

## Four rules that recur

- **Tokenising.** A search with no whitespace is one keyword, untouched (`ds:136-137`). Otherwise
  `split_keywords(…, keepquotes)` (`ds:139`): with a colon anywhere, split on whitespace only,
  keeping quotes, `;`, `|`, `*` (`sf:2101-2104`); without a colon, `cleanse_string` turns every
  `$config_separators` character and `,` `:` into spaces (`sf:2133`, `2179-2204`).
- **Re-quoting.** An unquoted token holding any of `! @ # $ % ^ & ( ) _ + - = [ ] { } ' " \ , . < > / ?`
  is re-quoted (`dsk:270-289`) and handled by the phrase branch (`dsk:758-848`), where
  `cleanse_string` makes the special character a space and an empty word is swallowed by `$noadd`
  (`dsk:780-782`). Phrase words are resolved without stemming (`dsk:784`) and an unknown word is
  **created** in `keyword` (`dsk:787`).
- **Wildcard branch** (`dsk:585-716`): the asterisk-stripped keyword is created if absent
  (`dsk:427-430`); fragments are probed against InnoDB's default stop-word list (`587-610`); then
  **full text** `MATCH(name) AGAINST('+kw' IN BOOLEAN MODE)` over `node` if no leading `*`, ≥ 3
  characters, none of `-+@<>().;`, no stop-word hit (`611-630`); else **LIKE** on `keyword.keyword`
  if the keyword minus `*` is word characters only (`631-646`); else **RLIKE** on `node.name` with a
  MATCH prefilter (`647-702`). Field restriction and hidden-field exclusion apply to all three
  (`502-519`).
- **What reads node names directly** (indexed or not): the full-text box (`dsk:38`), the wildcard
  full-text and RLIKE forms, date terms (`151-158`, `195-201`), `basic*` (`167-169`, `857-858`),
  numrange (`226-245`). Everything else — plain keywords (`717-731`), phrases (`807-820`), the LIKE
  wildcard form (`634-646`) — goes through `node_keyword`/`keyword`, which only `keywords_index`
  fields populate (`nf:121-128`, `145-147`), plus partial-index infixes (`sf:2270-2298`) and
  complete-index whole values (`nf:1082-1085`). "Visible" = `get_hidden_indexed_fields()`
  (`rf:7871-7893`): any named field that is inactive or not viewable, index flags ignored.

## Wildcards (SA5, SB3, SA7, SK1, SA8)

| Row | Core's actual behaviour | Claim | Wrong reason? | Proposed |
|---|---|---|---|---|
| A12 `*scape` | **LIKE branch**: `k.keyword LIKE '%scape'` on the keyword table (`dsk:631-646`, `*`→`%` at 635) — reaches only keywords of indexed fields plus partial-index infixes; never a non-indexed value; matches the keyword's end, not a word boundary | **No** — not RLIKE over node names | yes: identical on data where every "…scape" word is in an indexed field | **must**, rule corrected: keyword-suffix LIKE over indexed fields (fixture: value only in a non-indexed field → 0; in a partial-indexed field → infix match) |
| A13 `sc*pture` | full text `+sc*pture` (`dsk:611-630`); InnoDB's lexer reads `+sc*` as a required prefix and `pture` as an optional term — the part *before* `*` is honoured, the reverse of the claim; core's own `sc*` never takes this branch (2 chars → LIKE on keywords) | partly | the 69,679 (> `sculpt*`) fits only this parse | **no** — a tokenizer accident; decline (optional live check: `sc*zzzz` = `sc*pture`) |
| A15 `land*;sun*` | single token keeps `;` (`ds:136`); `;` fails full text and LIKE → **RLIKE** `\b(land.*?\|sun.*?)\b` with a MATCH prefilter (`dsk:647-702`, `sf:3265-3277`) over every visible node name; keyword "land;sun" created | yes | the form only exists for a single-token or field-prefixed search: with whitespace `;` becomes a space and the OR collapses to ANDed words | **must** for the single-token / `field:` form; fixture must show the whitespace collapse |
| B7 `title:*scape` | A12 restricted to the field: `LIKE '%scape' AND resource_type_field = <title>` (`dsk:513-518`, `631-646`) | **No** — LIKE on keywords, not RLIKE | yes: title is indexed | **must**, corrected; live `camera:*ikon` = 0 proves LIKE |
| H22 `sculpt*, land*` | two full-text unions `+sculpt*`, `+land*` ANDed (`dsk:611-630`, `dsu:62-73`) | yes | no | **must** |
| K2 `nikon*` | full text over every visible node name, no index flag on the path (`dsk:611-630`, `504`); side effect: keyword "nikon" created, so a later `nikon` resolves and returns empty rather than a suggestion | yes | consistent (= K3, K4) | **must**; fixture must not assert the suggestion text after a wildcard run |
| K3 `camera:nik*` | full text restricted to the field (`dsk:513-514`, `622-630`); `camera:ni*` (2 chars) would take the LIKE branch and find nothing; `camera:nikon` → standard union on `node_keyword` → empty | yes | the prefix sits exactly at the 3-character threshold | **must**; fixture: `nik*` matches, `ni*` and `nikon` do not |
| K4 `"@FULL_TEXT:nikon"` | the advanced-search box builds it (`sf:242-248`); MATCH over node names of every field not hidden (`dsk:29-49`); no `+`, so several words are any-word; a value containing `@` is wrapped as a phrase | yes | no | **must** |
| K8 `"@FULL_TEXT:[QUOTES]library book[QUOTES]"` | phrase MATCH over node names (`dsk:38`), inactive fields excluded as hidden (`rf:7883-7888`) | yes | a total of 1 shows the inactive field's exclusion only if its value sits on a different resource | **must**; fixture with the phrase on two resources |
| K9 complete-indexed `field:two words` | the whole value is one keyword at position 0 (`nf:1081-1085`), cut at 100 bytes (`sf:2228`); the test string splits into `field:two` AND `words`, which tests nothing about the field; reachable by `field:abc123` exact, `field:two*` (full text), `field:*words` (LIKE on the whole-value keyword) | partly | not measured | **must** — whole-value keyword; fixture as listed |
| K10 word past 500 chars | `mb_substr(…, 0, 500)` before splitting (`nf:1114-1115`); the full-text and RLIKE forms read the whole `node.name`; the LIKE form reads capped keywords | partly: "a wildcard reads the whole value" only for full text and RLIKE; `*ord`, `wo*` and stop-word-prefixed wildcards are capped | not measured | **must** — fixture: unique word at character 600: `word` 0, `wor*` 1, `*ord` 0, full-text box 1 |
| K11 `field:numrange1\|10` non-indexed | numrange branch for `type == 0` only (`dsk:205-246`), node join, no index flag; string→double coercion | yes | not measured | **must**; fixture only |
| K12 hidden field `field:word` | `metadata_field_view_access` false → `return false` before the hook (`dsk:121-124`, `ds:297-300`); its values are excluded from every union by `get_hidden_indexed_fields()` (`ds:286`, `dsk:40-43`, `504`, `506`, `797-800`); a field with an empty name is never hidden | yes | not measured | **must** |

Also established: a trailing wildcard (`sculpt*`) is full text over visible node names with no
candidate limit in SQL; `$max_results` is unused; related keywords never apply to wildcards (the
`$relatedsql` is used only by the standard union); `$wildcard_always_applied` appends `*` to every
keyword. A plugin hook `excludefieldsfromkeywordsearch` is broken for wildcards (`dsk:504` assigns
instead of appending) — a core defect for Q11.3.

## Parse rules and specials (SB2, SB5, SA3, SA4, SJ1, SF1, SF3, SF8, SF9, SF10, SF13, SF16)

| Row | Core's actual behaviour | Claim | Wrong reason? | Proposed |
|---|---|---|---|---|
| B3 `"title:henry moore"` | the advanced search sends it (`sf:357`); field parsed inside the quotes (`dsk:82-99`), words resolved without stemming, adjacency join `position + 1` restricted to the field (`dsk:758-845`) | yes | 251 = B2, so adjacency is unproved | **must**; fixture "Moore, Henry" (B2 matches, B3 must not) |
| B4 `title:"henry moore"` | tokens `title:"henry` and `moore"`; each re-quoted; the stray quote becomes a space and an empty word is swallowed; henry in title AND moore anywhere through the phrase path (no stemming, no related keywords) | partly — "quotes stripped" is really re-quote plus swallow | 251 = B2 only while stemming is off and no related keywords exist | **no** — accidental parse; decline |
| B17 `title:-sculpture` | re-quoted; phrase branch with `$omit`; `resolve_keyword('-sculpture')` **creates** a keyword no node maps to; `NOT IN` against it excludes nothing (`dsk:769-787`, `827-836`) | partly — not "omitted": a NOT against a junk keyword | everything is also what dropping the token gives; the junk keyword row is the trace | **bug** — NOT inside a field term silently ignored and pollutes `keyword`; decline meanwhile |
| I3 `title:sculpture,landscape` | re-quoted; `cleanse_string` splits on `,`; adjacency phrase in the field | yes | 0 is also what ANDed title terms give | **bug** — the field form of I2 (Core 10); same disposition as I2 |
| I6 `sculpture:` | unknown field → `explode(':')` appends `sculpture` and `''` to the keywords (`dsk:105-115`); `''` skipped | yes | = A1 by construction | **must** — deterministic |
| I7 `:sculpture` | the same path; **hazard:** core tolerates a field with an empty name, so on a database with one, `:sculpture` searches that field or returns false | yes on the test system | the total says the test DB has no nameless field | **must**, provided the plugin's field lookup is literal like core's |
| I8 `"sculpture` | single token; re-quoted because the leading `"` is a special character; phrase branch; quote dropped | partly | = A1 only while stemming is off | **no** — accidental path; decline |
| I12 `"source:igital Camer"` | `sf:280-291` builds it for a posted `field_<ref>` with a space, but dropdowns post `nodes_searched[<ref>]` (`render_functions.php:633-636`, `705`, `747`) and nothing implements `customchkboxes`; unreachable | yes | — | **n/a** |
| J4 `!contributions9,title:park, sculpture` | only text after the first space survives (`ds:125-133`) → `sculpture`; `$cuser` = `9,title:park,`, bound as `i` and coerced to 9 by mysqli (`sf:1438-1458`); `title:park` never becomes a keyword | yes, plus the coercion | the nonzero total corroborates coercion to 9 | **must**; the plugin's ref parse takes the leading integer |
| J9 `sculpture, @@6125, @@6130` | each token its own bucket (`ds:519-545`), one join per bucket (`dsn:17`) → AND whatever `$category_tree_search_use_and_logic` is; the form actually sends a double comma, removed by `ds:529-530` | yes | 0 is also what a nonexistent node gives (`metadata_field_view_access(0)` is true) | **must**; live: each node alone > 0 |
| A17 `the` | `$noadd` skip (`dsk:325`), no union, everything visible; **caveat:** the no-whitespace path never lowercases and the test is case-sensitive, so `The` is *not* skipped — it resolves as a keyword and gives a suggestion | yes | — | **must**, with the case rule copied exactly (live check `The`) |
| A30 `sculptre` | unknown word → suggestion before the hook (`dsk:387-400`, `do_search_suggest.php:4-28`); the test string is a keyword on the test DB because any field-restricted or quoted search with a misspelling creates it | mechanism yes; string no | exactly this | **n/a** (core answers first); replace the string with one that cannot be a keyword and run it before any B/I row uses the word |
| A10 `-"sculpture park"` | quoted omit: `r.ref NOT IN (phrase chain)` appended to the filter (`dsk:765-836`) | yes | 100,862 − 93,979 = A7's phrase count; adjacency vs co-occurrence unproved | **must**; fixture "park sculpture" not excluded |
| F69 `!last100, sculpture` | keyword after the space; `explode(',')[0]` → 100 (`sf:1127-1132`); inner `ORDER BY ref DESC LIMIT 100` | yes | any search with ≥ 100 matches returns 100 | **must**; live: refs compared with F1, or a rare word |
| F37 `!unused` | `r.ref NOT IN (SELECT resource FROM collection_resource)` — any collection, private and selection included (`sf:1464-1479`) | yes | none | **must** if memberships are indexed (they are, with every collection type — Q5.7 decides which kinds) |
| F39 `!listall…` | prefix `!list` matches `!listall` everywhere (`sf:825`, `1484-1490`); identical | yes | = F38 by construction | **must** — treat as `!list` |
| F62 `!propertiesbogus:1` | no `case` matches, `$propertiessql` empty, everything visible (`sf:1524-1618`) | yes | same total as the unknown-special fallthrough | **must** — unknown property names ignored inside a real property search (fixture: `bogus:1;fext:jpg` = `fext:jpg`) |
| F65 `!noningested` non-admin | not special for a non-admin (`sf:1629`) → standard query, everything; for admins the SQL applies **no `$sql_filter` at all** | yes, plus the unfiltered admin half | indistinguishable from F67 | **bug** — F67's; decline meanwhile |
| D16 `startdate:…, enddate:…` | nothing posts them; via the API they split into unknown words → a suggestion | yes | — | **n/a** |
| F48 `!empty170` restypes=1 | the inner type clause is correlated and inert (F47's bug); what limits the result is `search_filter()`'s `resource_type IN (1)` (`sf:751-758`); a non-global field with no type overlap gives `IN ()` — a SQL error | partly — right outcome, wrong mechanism | yes | **must** (type filter by construction); DB: is field 170 global? |
| F52 `!propertiesfext:jp*` | `*`→`%`, `r.file_extension LIKE 'jp%'`, collation-driven (`sf:1566-1578`) | yes for the mechanism | "jpg and jpeg" is only this data: `jp2`, `JPG` match too | **must**, expectation written as the LIKE |
| F59 `!propertiesorientation:portrait` | `COALESCE(rdim.height,0) > COALESCE(rdim.width,0)` via `LEFT JOIN resource_dimensions` (`sf:1588-1606`); no dimensions row counts as **square** | yes | the no-dimensions case is invisible here | **must**; fixture with a dimension-less resource |
| F63 `!integrityfail` | exact string match only (`!integrityfail sculpture` silently runs as a plain keyword search); `integrity_fail=1 AND no_file=0`; no archive clause but the pending and z filters still apply | partly | 0 exercises nothing | **no** — maintenance flags set outside the indexing pipeline; decline |

## Dates, numeric, option text (SD1, SD3, SD4, SD5, SE1, SC2, SC3)

| Row | Core's actual behaviour | Claim | Wrong reason? | Proposed |
|---|---|---|---|---|
| D4 `date:2017-05-10` | `dn1.name like '2017-05-10%'` string LIKE (`dsk:146-158`); index flag irrelevant; timed values `YYYY-MM-DD HH:MM` match | yes | the total cannot show any value carries a time | **must**; fixture with a timed value |
| D5 range | `drn1s.name >= '2024-01-01'` and `drn1e.name <= '2024-02-29 23:59:59'` as strings (`dsk:179-203`); one node per plain date field, so both joins hit it | yes | stored partials: `2024-02-00` inside, `2024-00-00` outside | **must** (Q3.4 (a) reproduces it); fixture with a `-00` partial and a timed end-day value |
| D7 end only | `$rangestart` unset; end join only. **Quirk:** `$rangestart` / `$rangeend` are never reset between keywords (`dsk:183-190`): a later range term inherits an earlier one's bound | yes | — | **must**; add a row for the stale-bound quirk (`date:rangestart…, date:rangeend…`) |
| D8 EDTF `2024-00-00` … `2024-12-99` | the EDTF box pads to `-00` / `-99` (`sf:3334-3344`); string compare; the padded year-only value `2024-00-00` equals the lower bound and is included | yes | equality with D1 is expected, so `-99` is not tested on its own | **must**; fixture with a `2024-12-31 23:xx` value |
| D9 `…end2024-02-31` | the form pads a blank day with `-31` (`sf:3369`); lexical compare makes it work | yes | = D5 as predicted | **must** |
| D22 non-indexed date `datephototaken:2024` | the date branch never reads `keywords_index`; nodes exist for every value | yes | the total would be the same if indexed; the code proves it | **must**; fixture only |
| D23 full date, timed values | as D4 | yes | shared seeded date, not timed values | **must** |
| D25 date-range field | **the rule is overlap, not containment:** a range resource has two nodes of the field, nothing distinguishes start from end, so the two joins mean *some node ≥ S and some node ≤ E* (`dsk:179-203`, `def:78-79`, `rf:856-857`): a `2019-06 / 2020-06` range and an enclosing `2019 / 2023` both match; the code comment (`dsk:192`) is a third, wrong reading; core's own test uses a plain date field | **No** | a total of 1 cannot separate overlap from containment | **must** with the rule corrected to overlap (two `_k` values per resource; `_k:>=S && _k:<=E` over an array gives exactly this); fixture: partial overlap, enclosing, outside, open-ended |
| D27 `date:2024;2025` | the date branch runs before the OR handling; `like '2024;2025%'`; nothing | yes | — | **no** — no form emits it; declining gives core's empty answer |
| J13 value and range on one field | three inner joins ANDed (`dsk:154-158`, `195-201`, `ds:399-412`) | yes (three joins, not two) | 524 vs 572 is explained by `2024-05-00` partials matching the LIKE but sorting before the range start — evidence of the string compare | **must**; fixture with a `2024-05-00` value |
| E6 `numberfield:42` | an ordinary keyword search restricted to the field (`dsk:291-296`, `516`); needs the field indexed; keyword "42" also matches `42.5` and `42-1` (separator split) and misses `042` | yes | = E2 only on this data | **must** (it is SB1); fixture with `42.5`, `042` |
| E7 `title:numrange1\|10` | accepted for any type-0 field, no `field_constraint` check (`dsk:205-210`); `rnn1.name >= ? AND <= ?` with `d` parameters → MySQL casts the text to double (non-numeric → 0) (`dsk:230-245`) | yes | 2,937 titles casting into 1..10 is plausible, unverified | **no** — the form only emits numrange for `field_constraint = 1` fields (`render_functions.php:540-578`); incidental casting; decline |
| H16 keyword + numrange | keyword union + numeric join ANDed (`dsu:38-76`) | yes | 0 is weak (a SQL error also shows as 0 through the API) | **must**; fixture with a resource holding both |
| C4 `usagerights:social` | exact option name fails (`dsk:262`) → keyword search restricted to the field (`dsk:291-296`, `516`); needs the field indexed | yes | = C2 only because one option contains the word | **must**; fixture with two options sharing a word |
| C5 `usagerights:soc*` | **full-text prefix on option names restricted to the field** (`dsk:611-630`), not LIKE on keywords; works on non-indexed fields; `so*` (2 chars) would take the LIKE branch | **No** — mechanism wrong | yes: both mechanisms return the same node here | **must**, rule corrected; fixture: 2-letter wildcard, non-indexed fixed-list field |
| C7 two options ANDed | with `$checkbox_and` (`sf:318-324`), two AND buckets (`dsn:17-18`); on the test DB the BOM name fails the exact match and falls to the keyword path; the index strips U+FEFF, so the keyword still maps to the BOM'd node | yes, BOM aside | 0 follows from several causes; two buckets were never exercised here. **Also:** C6's `bronze;wood` silently drops an alternative that is not an option name once another matches (`dsk:264-268`) | **must**; fixture with clean names and resources holding one, both, neither |
| C12 `artistname:moore` | dynamic-keyword nodes are all candidates; not an exact name → keyword path restricted to the field | yes | = C11 because one node contains the word | **must**; fixture with two options sharing a word |
| C23 option name + node of the same field | `@@388` bucket 0 (`ds:519-545`), `landscape` bucket 1 (`dsk:259-265`); two inner joins → both needed | yes | 0 shows 388 is not the landscape node, not that it is an orientation node | **must**; fixture confirming 388's field |

Also noticed: option-name matching sees only top-level nodes (`nf:333`, `$recursive = false`), so
tree children are unreachable as `field:name`; a wildcard field search creates the bare keyword as
a side effect (`dsk:427-429`).

## Parameters and sorts (D1, D5, D6, D9, SG1, SG2, SG4, SF2, SF6, SH1)

| Row | Core's actual behaviour | Claim | Wrong reason? | Proposed |
|---|---|---|---|---|
| G8 archive=2 with z2 | `archive IN (2)` plus, because `$uploader_view_override` defaults on, `(archive NOT IN (1,2,3) OR created_by = U)` (`sf:887-894`) — the user's **own** state-2 resources are visible | partly: "nothing" only if the override is off | 0 is consistent with both | **must** — D5's z exclusion must include the uploader override; fixture with an own state-2 resource |
| G9 archive=-1 | `archive IN (-1)` plus the pending filter `(archive<>-1 OR created_by = U)` (`sf:854-869`); `ert<type>` widens it | yes | 0 is also what a missing pending filter gives | **must**; fixture with own and others' -1 resources |
| G37 go | only the keyword-usage logging gate (`dsk:749-751`) and a hook argument; core still logs from `$keywords_used` when a plugin answers | yes | — | **n/a** |
| G38 editable_only | the edit filter (`sf:928-1062`) plus the group's edit filter; **core bug:** the `$additional_archive_states` loop tests `e<n>` with a stale `$n` (`sf:953-956`), so custom states are never blocked and state 4 is blocked instead | yes, plus omitted clauses | — | **no** until Q6.4; the bug to Q11.3 |
| G39 access | `r.access = ?` only for `v` users (`sf:920-926`); never passed by the API bindings; pages only | yes | cannot be exercised by the test users | **n/a** for the API; by construction with the `v` gate |
| G40 smartsearch | `count($archive)==0 \|\| ($archive_standard && !$smartsearch)` → defaults (`sf:833-839`); **`$archive_standard` is a page global** set by `pages/search.php:331-332`, unset elsewhere, so `$smartsearch` does nothing outside a search-page request; it exists so `update_smart_collection()` inside a search page keeps the saved states | yes, with that context | — | **n/a** via API; **D5 must read `$archive_standard`** when reproducing `search_filter()` |
| G41 ignore_filters | gates the date, numrange and fixed-list conversions (`dsk:128`, `205`, `247`); every core caller passes false; the only `true` callers (openai_gpt) use `!hasdata` or empty searches | yes — dead | — | **n/a** |
| G42 shapes | refs-only: same SQL, mapped to refs for scalar `fetchrows`, full rows for array `fetchrows` (`ds:430-440`); **special searches drop `ORDER BY` for refs-only** (`sf:1770-1780`); disk usage: aggregate wrapper (`ds:177-180`); returnsql: the query object, bypassing the share-key check; access_override: skips rca joins, `T`, confidential, the whole archive block and the z filter (`sf:801-886`) | yes | — | refs-only **must** (note the unordered special case); disk usage and returnsql **no**; access_override **must** by construction |
| G11 relevance | `score desc, user_rating desc, total_hit_count desc, field12 desc, r.ref desc` (`sf:3180`). `score` for a keyword search = `h.score` = `sum(score)` across union rows (`dsu:39`, `79`), where a plain keyword union contributes a **non-aggregated** `resource_node.hit_count` (`dsk:722-729`, legal because core strips ONLY_FULL_GROUP_BY); `resource_node.hit_count` is raised only when a resource is viewed from a search containing `@@` tokens (`view.php:233`, `rf:7482-7493`), copied daily; no keywords → `r.hit_count` (`ds:402-404`) | yes | G11 and G12 return the identical top five, so `score` is constant on the test DB | **must**; fixture with a non-zero node hit count on a matched node |
| G12 popularity | `user_rating desc, total_hit_count desc, field12 desc, r.ref desc` (`sf:3181`); `user_rating` = `ceil(avg)` from `user_rating_save()`, NULL unless `$user_rating` is on | yes | with ratings NULL the key is `r.hit_count, field12, ref` | **must**; fixture with rated resources |
| G13 rating | `r.rating desc, user_rating desc, score desc, r.ref desc` (`sf:3182`); **`resource.rating` is never written by core** from a field — only `copy_resource` and the merge tool copy it; `$orderbyrating` only adds the dropdown entry | yes for the SQL; the note was incomplete (ref order follows only because `score` also ties) | yes | **must**, treating `r.rating` as a legacy constant; DB: `COUNT(*) WHERE rating IS NOT NULL` |
| G16 title | `field8 desc, r.ref desc` on the copy column `resource.field8`, `VARCHAR(200)` — values truncated at 200, column collation | yes | no | **must**; live collation check |
| G17 file_path | `file_path desc, r.ref desc`; populated only for non-ingested / static-synced resources | yes | the list equals G18: every row ties on NULL | **no** — unexercised, admin-only key; core answers |
| G19 resourcetype | `rty.order_by desc, resource_type desc, r.ref desc` (`sf:3188`) | yes | ties indistinguishable | **must**; DB: `resource_type.order_by` |
| G20 extension | `file_extension desc, r.ref desc`; collation; NULL last | yes | no | **must** |
| G21 status | `archive desc, r.ref desc` | yes | every row ties on `archive=0`; = G18 | **must**; rerun with `archive=0,8` |
| G25 field12 | any `field…` passes the entry test; join check; `field12 desc, r.ref desc` (`sf:3245`) unless `sort_method=1` or `field_constraint=1` — identical to the date sort | yes | no | **must**; DB: field 12's `sort_method`, `field_constraint` |
| G27 field8 | as G25 → identical to G16 | yes | no | **must** |
| H11 collection order | `c.sortorder asc, c.date_added DESC, r.ref asc` (`sf:3196-3207`); keyword union inside the collection SQL | yes | the five refs are *descending*, which `r.ref asc` cannot produce: `sortorder` or `date_added` decided; the key is not shown to work | **must**; DB: `sortorder`, `date_added` of 3808's members |
| H28 list + node | node join (`dsn:17`) plus `r.ref IN (…)`; restypes would apply, archive parameter ignored, pending and z filters apply | yes | total 1 = F38: the node join may be a no-op | **must**; DB: node 389 on the three refs |
| H29 hasdata + node | node join then the `!hasdata` RIGHT JOIN (effectively inner); no archive clause, pending and z apply | yes | compare with `!hasdata170` alone | **must** |
| H38 collection + types + state | `resource_type IN` skipped and the archive parameter ignored for `!collection` (`sf:751`, `825-830`); pending and z filters still apply | partly — "states ignored" means the parameter | shown only if 3618 holds type-5 or non-state-0 members | **must**; DB: 3618's members by type and state |

**H31, the capstone**, expectation: type 1, state 0, carries node 389 (twice, once as the token and
once as the resolved option — harmless except that `total_hit_count` sums the node twice), has the
keyword "sculpture" in any visible indexed field, has "sculpture" immediately followed by "park" in
one field, field 12 beginning `2024`, and no field with "black" followed two positions later by
"white" ("and" is a `$noadd` word, so the NOT phrase is `black _ white`); ordered by the field-12
text descending, then ref descending. Claim holds; **must**; the fixture must prove the NOT phrase
and the adjacency separately — the live 307 cannot.

## Consequences for decisions already taken

- **Q1.1 (b):** still right, for the right rows — the trailing wildcard (≥ 3 characters, no stop
  word) and the full-text box read every visible field's values; the **leading** wildcard does not
  (keyword table, indexed fields only). SA5 and SB3 need two mechanisms, not one: a prefix over all
  values for `word*`, and a keyword-suffix match over indexed fields for `*word` (a reversed-token
  representation or a decline — Q2.3).
- **Q3.4 (a):** confirmed for plain date fields. For the **date-range field type** the rule is
  overlap (SD4), which `_k` reproduces as an array: `_k:>=S && _k:<=E` over the resource's two values.
- **D5 must honour** `$uploader_view_override` in the z exclusion (own resources stay visible) and
  the page global `$archive_standard` when reproducing the archive defaults.
- **D9 / Q7.1:** core's keyword `score` is `resource_node.hit_count`, fed only by views from
  node-token searches and daily-copied — near zero on most systems, so core's relevance is in
  practice hit count, date, ref; the listed difference is smaller than feared.
- **G42:** a refs-only special search is unordered in core; the plugin must not promise order there.

## Core defects noticed in passing (for Q11.3 and I4)

`dsk:504` assigns instead of appending the hidden-field clause for wildcards (breaks the
`excludefieldsfromkeywordsearch` hook); `sf:953-956` stale `$n` in the editable-only state loop;
`dsk:566-567` `!empty` on a non-global field with no type overlap → `IN ()` SQL error;
`sf:1619` `!integrityfail <word>` silently a plain search; `dsk:183-190` range bounds not reset
between keywords; `nf:333` option names matched at top level only; wildcard and phrase searches
create keyword rows (`dsk:427-430`, `787`); `dsk:264-268` `field:a;b` drops an unmatched
alternative; `pages/search.php:612`, `614` pass 19 arguments to an 18-parameter `do_search()`.

## Still needing a run

Live (light): `camera:*ikon` (A12/B7), `sc*zzzz` (A13), `The` (A17), the H11/H28/H38 data queries,
G21 with two states, F69's refs, J9's nodes alone. Fixture or H9: every "fixture" entry above.
