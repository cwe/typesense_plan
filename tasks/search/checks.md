# The check register

Every catalogue row marked *check* (77 rows: 78 on 2026-10-20, then F38 marked *must* the same day because J1 depends on it), grouped by what has to be confirmed in core
before the row's mark is final. A check is run against core itself — the live API runner on the test
system for rows its data can show, and the MySQL-backed harness mode (H9) for the rest — and its
outcome is recorded on the row: *must*, *no*, *n/a*, or *bug* (Q11.3). Until then the row keeps its
package and its Plan symbol in [coverage.md](coverage.md); a package with only check rows is
**pending check** and is not started.

Where a check could show that core treats the form as a search type of its own, the theme says so
(*split candidate*): that is the point at which a package is cut in two or a new one added.

Packages pending check (no *must* row): SD4, SD5, SF16, SG1, SG2, SG4, SH1, SK3. The pipeline tasks D1, D5, D6 and D9 own
check rows too (G37, G38, G42, G8, G9, G40, G39, G11, G25) but are required regardless.

## Wildcards beyond a trailing prefix

Core answers a leading or inner wildcard from a different branch than a trailing one: RLIKE over node names of every visible field, indexed or not (A12, B7, K2, K3), or the full-text branch where MySQL honours only the trailing part (A13). Confirm which branch each form takes, over which fields, and what `sculpt*, land*` does with two of them.

**Split candidate.** If the RLIKE branch is confirmed, non-prefix wildcards are a search type of their own (values of every visible field, not the keyword index): either a new package beside SA5 served from an all-values representation (Q1.1 option b), or a declared decline. H9 (the MySQL-backed harness mode) is the tool for this.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| A12 | Leading wildcard: `*scape` | RLIKE on node names (".*?scape" at a word boundary). | Q2.3 | SA5 |
| A13 | Wildcard inside a word: `sc*pture` | Goes to the full-text branch as "+sc*pture"; MySQL honours only the trailing part of the pattern, so this behaves like sc*. | Q2.17 | SA5 |
| B7 | Leading wildcard in a field: `title:*scape` | RLIKE restricted to the field. | Q2.3 | SB3 |
| H22 | Two wildcards: `sculpt*, land*` | Two full-text unions. | Q2.3 | SA5 |
| K2 | The same word with a wildcard: `nikon*` | The full-text branch reads node names of every visible field, indexed or not: the camera values match. | Q1.1, Q2.3 | SK1 |
| K3 | Wildcard inside the non-indexed field: `camera:nik*` | Full-text restricted to the field: matches, although camera:nikon (B12) does not. | Q1.1 | SK1 |
| C5 | Wildcard on an option: `usagerights:soc*` | LIKE on keywords within the field. | Q3.12 | SC3 |

## Negation forms

A10: confirm that a negated phrase excludes exactly the phrase matches. B17: confirm that `title:-sculpture` excludes nothing — if so it is a core bug and takes the *bug* mark (Q11.3).

No split: A10 stays in SA4; B17 is a decline or a bug.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| A10 | NOT a phrase: `-"sculpture park"` | Everything except resources with the phrase. | Q2.6 | SA4 |
| B17 | NOT after the colon: `title:-sculpture` | Quoted, then the word "-sculpture" is omitted: excludes nothing. | Q3.8 | SB5 |

## OR forms

Confirm the RLIKE OR of two wildcards and its result set on a fresh database, next to A16/H32 (Core 9).

No split: SA7's mechanism (Q2.7) serves it or declines it.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| A15 | OR of two wildcards: `land*;sun*` | Semicolon forces the RLIKE branch: "(land.*?\|sun.*?)". | Q2.7 | SA7 |

## Stop words and unknown words

Both are answered by core before the hook on the test system (everything / a suggestion). Confirm on a fresh database that the plugin is never asked.

No split; the rows are `?`/`·` by construction.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| A17 | Stop word alone: `the` | Skipped ($noadd); no terms left, so everything. | Q2.4 | SA2 |
| A30 | Misspelt word: `sculptre` | An unknown word gives no results and a sound-alike suggestion instead of an array; on the test system this misspelling happens to exist as a keyword, so A31 … | — | SA11 |

## Quoted, punctuated and malformed field terms

Each is a parse rule: the two quoted forms (B3 the supported one, B4 the stray-quote one), a comma inside a field term (phrase of the parts), a trailing or leading colon (the colon becomes a space), an unbalanced quote (a separator), and the unreachable dropdown branch. Confirm each rule as stated in the catalogue's *What core does*.

No new search type: these settle D2's parse table (Q2.8, Q3.2, Q3.18). I12 may be marked *n/a* if the branch is confirmed unreachable.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| B3 | Quoted phrase in a field: `"title:henry moore"` | Phrase search restricted to the field (the form the advanced search sends). | Q3.2, Q3.18 | SB2 |
| B4 | Quotes after the colon: `title:"henry moore"` | Not the supported form: the tokens are title:"henry and moore", but the stray quotes are stripped, so it works out the same as B2. | — | SB2 |
| I3 | Comma inside a field term: `title:sculpture,landscape` | Quoted because of the comma, then split: the phrase "sculpture landscape" in title. | Q2.8 | SB4 |
| I6 | Trailing colon: `sculpture:` | Field "sculpture" does not exist: colon becomes a space, so the word sculpture. | — | SB1 |
| I7 | Leading colon: `:sculpture` | Empty field name: same as I6. | — | SB1 |
| I8 | Unbalanced quote: `"sculpture` | Not a quoted string (no closing quote); the quote is a separator. | — | SA3 |
| I12 | Dropdown value from the advanced form: `"source:igital Camer"` | search_form_to_search_query has a branch that strips the first and last characters of a dropdown value with spaces; dropdowns post node refs instead, so the … | — | SJ1 |

## Option text core does not resolve

C4 and C12: one word of an option or dynamic keyword falls back to a keyword search inside the field — confirm, and whether the fallback honours the field's index flags. C7: confirm on clean data (the test system's byte-order mark is data). C23: confirm that an option name and a node of the same field are two ANDed buckets (0 on the test system).

**Split candidate.** If the keyword-in-field fallback is confirmed, SC3 is really two forms: an option name that resolves (SC2) and free text inside a fixed-list field (served from the option names `_ss`, Q3.12) — the latter may deserve its own package if the wildcard case (C5) is also in.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| C4 | One word of an option: `usagerights:social` | Not an option name, so a keyword search restricted to the field. | Q3.12 | SC3 |
| C7 | AND of options: `materials:bronze, materials:wood` | Two buckets: resources with both (the $checkbox_and form). On the test system the first option's name is stored with a leading byte-order mark, so it is not … | Q3.12 | SC3 |
| C12 | One word of a dynamic keyword: `artistname:moore` | Keyword search in the field. | Q3.12 | SC3 |
| C23 | Option name and a node of the same field: `orientation:landscape, @@388` | Two buckets on one field: both options needed. | — | SC2 |

## Dates

D4/D23: values stored with a time of day match a date prefix — confirm the prefix rule. D5, D7, D8, D9: the range forms are string comparisons in core (end padded to 23:59:59, EDTF `-00`/`-99`, a day padded to 31) — confirm each bracket and what falls outside it. D16: confirm the inputs are dead. D22: a date term on a non-indexed field works because dates join the node table — confirm. D25: a date-range field's range is containment (start ≥, end ≤), not overlap — confirm. D27: `;` on a date gives nothing — confirm. J13: a value and a range on one field are ANDed — confirm.

**Split candidates.** (1) The string-comparison ranges (D5–D9) are either reproduced as string comparisons on a canonical key or declined (Q3.4's direction); (2) date-range fields (SD4) are containment, a different rule from SD3's ranges; (3) non-indexed date fields (SD5) need `_q` regardless of the flag (Q3.6). Each could stand as its own package if the check confirms the rule.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| D4 | Date stored with a time: `date:2017-05-10` | The prefix still matches a value stored with a time of day. | — | SD1 |
| D5 | Range: `date:rangestart2024-01-01end2024-02-29` | name >= start AND name <= "end 23:59:59". | Q3.4 | SD3 |
| D7 | Range, end only: `date:rangeend2005-12-31` | name <= end. | Q3.4 | SD3 |
| D8 | Range as the EDTF box sends it: `date:rangestart2024-00-00end2024-12-99` | "2024/2024" in the EDTF box becomes this; string comparison still brackets the year. | Q3.4 | SD3 |
| D9 | Range, end month without a day: `date:rangestart2024-01-01end2024-02-31` | The form pads a missing day with 31; as a string "2024-02-31" is after every February date. | Q3.4 | SD3 |
| D16 | startdate / enddate form inputs: `startdate:2024-01-01, enddate:2024-12-31` | search_form_to_search_query still assembles these, but no page posts the inputs and nothing in do_search handles them: unknown field names, so words and a su… | — | SD2 |
| D22 | Date field not flagged for indexing: `datephototaken:2024` | Date terms join the node table directly, so the index flag does not matter. | Q3.6 | SD5 |
| D23 | Same field, full date: `datephototaken:2012-01-26` | Matches values stored with a time of day. | Q3.6 | SD5 |
| D25 | Date-range field, range: `daterangetest:rangestart2020-01-01end2021-12-31` | Start value >= start AND end value <= end, as strings. | Q3.5 | SD4 |
| D27 | OR on a date: `date:2024;2025` | No OR for dates: LIKE "2024;2025%", nothing. | Q3.14 | SD1 |
| J13 | Date value and range on one field: `date:2024-05, date:rangestart2024-05-01end2024-05-31` | The date boxes and the range boxes are separate inputs, so both terms are sent: two joins, ANDed. | — | SD3 |

## Numeric

E6: a plain number in a numeric field is an ordinary keyword search — confirm. E7: `numrange` on a text field compares text values as numbers — confirm and decide serve or decline. H16: keyword plus numeric join. K11: a range on a non-indexed numeric field works through the node join — confirm on the fixture (no such field on the test system).

**Split candidate.** E7 (numeric comparison of text values) is a form of its own; if it is wanted it needs a numeric representation of text fields, otherwise it is a declared decline.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| E6 | Plain number in the field: `numberfield:42` | Not a range: an ordinary keyword search for "42" in the field. | — | SE1 |
| E7 | numrange on a text field: `title:numrange1\|10` | Accepted for any type 0 field: compares title values as numbers. | Q3.7 | SE1 |
| H16 | Keyword and numeric range: `sculpture, numberfield:numrange0\|100` | Union plus numeric join. | — | SE1 |
| K11 | Number range on a non-indexed numeric field: `field:numrange1\|10` | Node join, so it works regardless of the flag, as dates do (D22). No such field on the test system. | Q3.6 | SE1 |

## Special searches

`!unused` (F37), `!list`/`!listall` and their state and z-permission handling (F39; F38 is *must* because J1 depends on it), `!empty` with a resource type (F48), the `!properties` keys (F52 extension wildcard, F59 orientation from the dimensions table, F62 unknown key → everything), `!integrityfail` (F63), `!noningested` for non-admins → everything (F65), `!last` with a comma (F69), and the collection, list and hasdata combinations (H11, H28, H29, H38). Confirm each against the catalogue's *What core does*.

**Split candidate.** `!properties` keys behave as three families (a text property with wildcards, a computed one from dimensions, an unknown key that filters nothing): SF9 may split by family. F65 and F62 ("everything") are decline-by-construction candidates.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| F37 | !unused: `!unused` | In no collection. | Q4.7 | SF16 |
| F39 | !listall: `!listall416:4460:8692` | Same, by name "all"; the filter difference is in search_filter, which treats both alike. | — | SF3 |
| F48 | !empty with a resource type: `!empty170` restypes=1 | Candidates limited to the given types. | Q4.8 | SF10 |
| F52 | !properties, extension wildcard: `!propertiesfext:jp*` | * becomes %: jpg and jpeg. | Q4.1, Q4.11 | SF9 |
| F59 | !properties orientation: `!propertiesorientation:portrait` | Compares resource_dimensions height and width. | Q4.1, Q4.11 | SF9 |
| F62 | !properties, unknown property: `!propertiesbogus:1` | No filter added: everything. | Q4.11 | SF9 |
| F63 | !integrityfail: `!integrityfail` | integrity_fail = 1 and a file present; all workflow states. | Q4.1 | SF13 |
| F65 | !noningested (admin only): `!noningested` | For admins: resources with a file_path. For anyone else no branch matches, so a plain search with no terms: everything. | Q4.1, Q4.9 | SF13 |
| F69 | !last, comma before the keyword: `!last100, sculpture` | The comma ends the number, so the last 100 that match (compare F4). | — | SF1 |
| H11 | Collection and keyword, collection order: `!collection3808 sculpture` order_by=collection, sort=asc, fetchrows=0,5 | Keyword inside the collection, collection order. | — | SF2 |
| H28 | List and node: `!list416:4460:8692 @@389` | Ref list plus node join. | — | SF3 |
| H29 | Hasdata and node: `!hasdata170 @@389` | Field join plus node join. | — | SF6 |
| H38 | Resource types and collection ignored together: `!collection3618 sculpture` restypes=5, archive=8 | Types and states are both ignored for a collection search; the keyword still applies. | — | SF2 |

## Sorts

Confirm core's order expression for each (the catalogue lists them) and, for G12/G13, what `user_rating`, `hit_count` and `r.rating` hold on a live system and how they move. G25: a date field by number is the date sort. G27: a text field by number needs a per-field key. G16: collation equivalence for the title key.

**Split candidate.** SG1 (text keys), SG2 (count-based keys) and SG4 (field by number) are already separate; the check decides which are built and in what order (Q7.2). None is required until a row is marked *must*.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| G11 | Order: relevance (default): `sculpture` order_by=relevance, fetchrows=0,5 | score, user_rating, hit count, date field, ref. | Q7.1 | D9 |
| G12 | Order: popularity: `sculpture` order_by=popularity, fetchrows=0,5 | user_rating, hit count, date, ref. | Q7.2 | SG2 |
| G13 | Order: rating: `sculpture` order_by=rating, fetchrows=0,5 | r.rating, user_rating, score, ref. | Q7.2 | SG2 |
| G16 | Order: title: `sculpture` order_by=title, fetchrows=0,5 | $view_title_field, ref. | Q7.2 | SG1 |
| G17 | Order: file_path: `sculpture` order_by=file_path, fetchrows=0,5 | file_path, ref. | Q7.2 | SG1 |
| G19 | Order: resourcetype: `sculpture` order_by=resourcetype, fetchrows=0,5 | resource type order_by, type, ref. | Q7.2 | SG1 |
| G20 | Order: extension: `sculpture` order_by=extension, fetchrows=0,5 | file_extension, ref. | Q7.2 | SG1 |
| G21 | Order: status: `sculpture` order_by=status, fetchrows=0,5 | archive, ref. | Q7.2 | SG1 |
| G25 | Order: a field by number (date): `sculpture` order_by=field12, fetchrows=0,5 | field12, ref (must be in the resource table joins). | — | D9 |
| G27 | Order: title field by number: `sculpture` order_by=field8, fetchrows=0,5 | field8, ref: text order. | Q7.9 | SG4 |

## Parameters and request modes

The z-permission exclusion (G8: nothing for these users), the pending-submission rule (G9: own resources only), `go` (logging only), editable-only (G38: the edit filter), `$access` for `v` users (G39), smart-search states (G40), `$ignore_filters` (G41: which callers set it), and the result shapes (G42). Confirm each in core; most are rules of D1, D5 and D6.

**Split candidate.** G38 (editable-only) is the one that is a search type of its own (Q6.4); the rest settle restriction rules.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| G8 | State hidden by a z permission: `sculpture` archive=2 | archive IN (2) AND archive NOT IN (1,2,3): nothing for these users. | — | D5 |
| G9 | Pending submission: `sculpture` archive=-1 | Only the user's own resources in state -1. | — | D5 |
| G37 | go (previous / next): `(any)` | Page direction for view.php; only affects keyword logging in do_search. | — | D1 |
| G38 | editable_only: `(any)` | Adds the edit-permission filter (e states, ert, XE, edit filter). | Q6.4 | D1 |
| G39 | access: `(any)` | r.access = n, only for users with v. | — | D6 |
| G40 | smartsearch: `(any)` | Keeps the saved archive states instead of the defaults. | Q5.11 | D5 |
| G41 | ignore_filters: `(any)` | Skips date, numrange and fixed-list conversion of field terms. | Q3.15 | D2 |
| G42 | return_refs_only / return_disk_usage / returnsql / access_override: `(any)` | Shape of the result, disk usage wrapper, SQL instead of rows, permission override for smart collections. | — | D1 |

## Combinations and page strings

H31: the capstone — confirm the exact set, total and order in core before the gate is armed. J4: only what follows the first space survives a posted special prefix — confirm. J9: `$checkbox_and` sends one token per option — confirm the AND.

No split: these confirm composition.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| H31 | Everything at once: `sculpture, "sculpture park", @@389, orientation:landscape, date:2024, -"black and white"` restypes=1, archive=0, order_by=date, sort=desc, fetchrows=0,5 | Keyword, phrase, node, option (same node twice), date, NOT phrase, type, state, order. | — | SH1 |
| J4 | Posted special search prefix, more terms: `!contributions9,title:park, sculpture` | Only what follows the first space is kept: sculpture applies, title:park is lost. | — | SJ1 |
| J9 | Checkbox list with $checkbox_and: `sculpture, @@6125, @@6130` | Each ticked option becomes its own token: AND. Both options resolve here, unlike the name form in C7. | — | SC2 |

## Indexing flags

The full-text box reaches non-indexed and active fields through node names (K4, K8) — confirm. Complete indexing makes the whole value one keyword (K9) and the 500-character cap (K10) — both need the fixture. A hidden field returns false before the hook (K12).

**Split candidate.** K4/K8 belong with SA8's full-text box, which now serves the box (A27–A29, A34 *must*): if the box must also reach non-indexed fields, SA8 needs the all-values representation of Q1.1 — the same split as the wildcard theme.

| Row | Search | What core does | Settles | Owner |
|---|---|---|---|---|
| K4 | Full-text box on a non-indexed field: `"@FULL_TEXT:nikon"` | MATCH over node names: matches. | — | SA8 |
| K8 | The same phrase through full text: `"@FULL_TEXT:[QUOTES]library book[QUOTES]"` | Node names are read: the active non-indexed field matches, the inactive one is excluded as hidden. | — | SA8 |
| K9 | Complete indexing: `field:two words` | The whole value is one keyword; a typed search splits on spaces, so only a single-token value is reachable by word, and wildcards or full text otherwise. No … | Q1.12 | SK3 |
| K10 | Word beyond the 500th character: `field:word` | Only the first $node_keyword_index_chars characters become keywords; a wildcard still reads the whole value (E10 in the review). The one indexed value over 5… | Q1.13 | SK3 |
| K12 | Field hidden from the user by permission: `field:word` | A field term on it returns false before the plugin is asked; its values are excluded from every union. Both test users can see every field, so not measured. | — | SK2 |

