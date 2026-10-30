# Catalogue coverage

Every row of the core search catalogue (`docs/core-search-catalogue.md` on `typesense_search_catalogue`, measured 6–9 October 2026) with the package that owns it and what the plan intends for it. The package files are in this folder; the pipeline tasks (D1–D11) are in [../D-search-pipeline.md](../D-search-pipeline.md). **Core** is the live total for the core user; **Old** is what the previous plugin returned (0 usually means it declined in Typesense-only mode). A row is owned by exactly one package; it may be a case in others.

Plan: `=` serve with core's result · `↩` decline, core answers · `≈` accepted difference (needs a decision row) · `✗` a core bug, disposition per Q11.3 · `·` nothing to build (core answers before the hook, or the form is unreachable) · `?` depends on the question named in the note. Rows marked *local harness* cannot be sent through the API and are covered by the local batteries only (Q10.14).

**Required** is the review mark copied from [../../catalogue.md](../../catalogue.md) (must / nice / no / n-a / bug); the Plan column was re-derived from them on 2026-10-20 and 2026-10-30 (see Totals). Status column: blank until the package's battery is green for the row.


## A — Keywords

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| A1 | One keyword: `sculpture` | 49,468 | 49,468 | SA1 | = |  | must |  |
| A2 | Two keywords: `sculpture landscape` | 34,408 | 34,409 | SA1 | = |  | must |  |
| A3 | Comma separated: `sculpture, landscape` | 34,408 | 34,409 | SA1 | = |  | must |  |
| A4 | Upper case: `SCULPTURE Park` | 19,680 | 19,688 | SA1 | = |  | must |  |
| A5 | Plural (stemming): `sculptures` | 49,468 | 49,468 | SA12 | = | stemmer drift Q2.14 | must |  |
| A6 | Quoted single word: `"sculpture"` | 49,069 | 0 | SA3 | ? | Q2.5 | must |  |
| A7 | Quoted phrase: `"sculpture park"` | 6,883 | 0 | SA3 | ? | Q2.5 | must |  |
| A8 | Phrase with a stop word: `"black and white"` | 3,179 | 3,179 | SA3 | ? | Q2.4 stop-word gap | must |  |
| A9 | NOT a keyword: `-landscape` | 32,045 | 32,078 | SA4 | = | the 33 extra are inactive-field keywords (SK2, Q6.7) | must |  |
| A10 | NOT a phrase: `-"sculpture park"` | 93,979 | 100,862 | SA4 | ? | Q2.6 negated phrase | must — fixture: "park sculpture" is not excluded |  |
| A11 | Trailing wildcard: `sculpt*` | 51,007 | 49,610 | SA5 | = | prefix over every eligible visible field (Q1.1 (b)); the trailing-wildcard mechanism and caps as H22 (2026-10-30) | must |  |
| A12 | Leading wildcard: `*scape` | 68,601 | 381 | SA5 | = | keyword-suffix match; Q2.3 settled by Q1.1 (b); representation scheduled with SA5 | must — rule corrected: LIKE '%scape' on the keyword table, so only keywords of indexed fields and partial-index infixes; needs the reversed-keyword representation (SA5) |  |
| A13 | Wildcard inside a word: `sc*pture` | 69,679 | 0 | SA5 | ↩ | declined — tokenizer accident (sc*zzzz = sc*pture live) | no — a tokenizer accident: InnoDB honours the part before the *, so sc*pture = sc*; declined, core answers |  |
| A14 | Short wildcard (under 3 letters): `so*` | 15,618 | 6,676 | SA5 | ? | Q2.16 keyword-table branch | must |  |
| A15 | OR of two wildcards: `land*;sun*` | 71,530 | 0 | SA7 | = | Q2.7 mechanism 4; whitespace collapse on the fixture | must — the single-token and field: forms; fixture must show the whitespace collapse |  |
| A16 | OR of two plain words: `sculpture;landscape` | 83,840 | 0 | SA7 | ✗ | check core on a fresh database first (Core 9: a suggestion until a field OR has created the keyword); an OR needs Q2.7's mechanism | bug — Core 9 (was check); core looks up the literal keyword "sculpture;landscape": a suggestion on a fresh database; it was an OR on the test system only because B5 had created that keyword. A real OR would be an improvement over core (Q2.7) |  |
| A17 | Stop word alone: `the` | 100,862 | 32,537 | SA2 | = | stop word dropped as core does; case rule exact | must — core drops the stop word and returns everything; case rule copied exactly (The ≠ the) |  |
| A18 | Stop word plus a word: `the sculpture` | 49,468 | 17,228 | SA2 | = | after stripping $noadd | must |  |
| A19 | Hyphenated word: `black-and-white` | 3,179 | 3,194 | SA6 | ? | Q2.8 | must |  |
| A20 | Apostrophe: `atkinson's` | 2 | 3 | SA6 | ? | Q2.8 | must |  |
| A21 | Related keyword: `antony` | 1,491 | 205 | SA10 | ? | Q2.10 | must |  |
| A22 | The related word itself: `anthony` | 1,491 | 1,300 | SA10 | ? | Q2.10 | must |  |
| A23 | Resource type name: `document` | 490 | 375 | SA9 | ? | Q2.9 / Q1.7 | must |  |
| A24 | Contributor name: `admin` | 13 | 16 | SA9 | ? | Q2.9 / Q1.7; option off on the test system | must |  |
| A25 | A resource number: `416` | 100 | 0 | SA9 | ? | Q7.4 exact ref first; Q2.11; ref_s matches the number as a word (Q1.10) | must |  |
| A26 | A year as a word: `2024` | 10,289 | 0 | SA9 | ? | Q2.11 year as a word | must |  |
| A27 | Full-text box: `"@FULL_TEXT:distant island"` | 956 | 0 | SA8 | = | Q2.15: the box is served; optional words by Q2.7 mechanism 4 (as A15), merged searches as the fallback | must — boolean mode: words without an operator mean any word (956), not all (9); optional words need one search per alternative, merged (SA8); words found only in non-indexed fields need Q1.1 option b |  |
| A28 | Full-text with operators: `"@FULL_TEXT:+distant +island"` | 9 | 0 | SA8 | = | Q2.15: required words map directly | must — required words map directly (SA8) |  |
| A29 | Full-text phrase: `"@FULL_TEXT:[QUOTES]distant island[QUOTES]"` | 8 | 0 | SA8 | ? | Q2.5 (the phrase, SA3); the box itself is settled (Q2.15) | must — a phrase (SA8, SA3) |  |
| A30 | Misspelt word: `sculptre` | 1 | 1 | SA11 | ↩ | core answers first; declined | no — an unknown word is answered by core before the plugin is asked; declined |  |
| A31 | Known plus unknown word: `sculpture zzzzqqq` | 0 | 0 | SA11 | · | marked must: already so — core returns the suggestion before the hook | must — already so: core returns the "did you mean" suggestion before the plugin is consulted; results for the known word would need a core change (Q2.12) |  |
| A32 | Only an asterisk: `*` | 100,862 | 100,862 | SA1 | = | empty search | must |  |
| A33 | Empty search: (empty) order_by=resourceid | 100,862 | 100,862 | SA1 | = | order by D9 | must |  |
| A34 | Full-text exclusion: `"@FULL_TEXT:+distant -island"` | 880 | 880 | SA8 | = | Q2.15: required and excluded words map directly | must — required and excluded words map directly (SA8) |  |

## B — Field-specific terms (text fields)

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| B1 | Word in a field: `title:sculpture` | 4,226 | 4,226 | SB1 | = |  | must |  |
| B2 | Two words after a field: `title:henry moore` | 251 | 251 | SB2 | = | by construction | must |  |
| B3 | Quoted phrase in a field: `"title:henry moore"` | 251 | 0 | SB2 | ? | Q3.2 / Q3.18 | must — fixture "Moore, Henry": B2 matches, B3 must not |  |
| B4 | Quotes after the colon: `title:"henry moore"` | 251 | 1 | SB2 | ↩ | declined — accidental parse | no — an accidental parse (re-quote plus swallow); declined, core answers |  |
| B5 | OR inside a field: `title:sculpture;landscape` | 5,120 | 0 | SA7 | = | Q2.7 mechanism 4: one filter clause with \|\| | must |  |
| B6 | Wildcard in a field: `title:sculpt*` | 4,522 | 4,522 | SB3 | = | prefix restricted to the field; max_filter_by_candidates raised and proved by SA5's battery (Q3.10 settled with H22, 2026-10-30) | must |  |
| B7 | Leading wildcard in a field: `title:*scape` | 921 | 0 | SB3 | = | A12 restricted to the field | must — rule corrected: LIKE '%scape' on keywords restricted to the field (camera:*ikon = 0 live proves it) |  |
| B8 | Another text field: `credit:wilde` | 18,879 | 18,879 | SB1 | = |  | must |  |
| B9 | Field name with a slash: `photographer/videographer:wilde` | 33,517 | 100,862 | SC3 | ? | Q3.12 one word of a dynamic keyword | must |  |
| B10 | Punctuation in a field value: `exhibitiontitle:fabric-ation` | 1,453 | 1,453 | SB4 | ? | Q2.8 / Q3.18 phrase of the parts | must |  |
| B11 | Dots in a field value: `dimensions:16.5x16.5cm` | 20 | 20 | SB4 | ? | Q2.8 / Q3.18 | must |  |
| B12 | Text field not flagged for indexing: `camera:nikon` | 0 | 0 | SK1 | = | Q1.1 (b): field terms filter flagged fields only — nothing, as core | must |  |
| B13 | Multi-line field not flagged for indexing: `accessibilityalttext:island` | 0 | 0 | SK1 | = | Q1.1 (b): field terms filter flagged fields only — nothing, as core | must |  |
| B14 | Partially indexed field, fragment: `originalfilename:ksho` | 665 | 665 | SB6 | = | _p | must |  |
| B15 | Unknown field name: `foo:sculpture` | 25 | 25 | SB1 | = | unknown name → two words | must |  |
| B16 | NOT before a field term: `-title:sculpture` | 49,241 | 0 | SB5 | ↩ | not required — Q3.8 degenerate form | no |  |
| B17 | NOT after the colon: `title:-sculpture` | 100,862 | 4,226 | SB5 | ✗ | NOT after the colon; Q11.3 | bug — NOT inside a field term excludes nothing and creates a junk keyword (I4, found 2026-10-22); was check; declined meanwhile |  |
| B18 | Stop word in a field: `title:the` | 100,862 | 10,564 | SB5 | ? | Q2.4 everything in core | must |  |
| B19 | Field hidden from advanced search, not indexed: `yearai:2025` | 0 | 0 | SK1 | = | Q1.1 (b): accepted by name, no flagged representation to filter — nothing, as core | must |  |
| B20 | Two terms, one field: `title:sculpture, title:park` | 580 | 580 | SB1 | = |  | must |  |
| B21 | Two fields: `title:sculpture, credit:wilde` | 316 | 316 | SB1 | = |  | must |  |

## C — Fixed-list fields and node tokens

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| C1 | Option by name: `orientation:landscape` | 58,028 | 58,028 | SC2 | = | resolved before the hook | must |  |
| C2 | Multi-word option, quoted: `"usagerights:Social media"` | 148 | 0 | SC2 | = | quoted token must be dropped from q | must |  |
| C3 | Multi-word option, unquoted: `usagerights:social media` | 148 | 4,763 | SC3 | ? | Q3.12 | must |  |
| C4 | One word of an option: `usagerights:social` | 148 | 100,862 | SC3 | ? | Q3.12 | must — fixture with two options sharing a word |  |
| C5 | Wildcard on an option: `usagerights:soc*` | 148 | 100,862 | SC3 | ? | Q3.12; mechanism corrected | must — rule corrected: full-text prefix on option names restricted to the field (2-letter wildcards take the keyword LIKE branch); works on non-indexed fields |  |
| C6 | OR of options: `materials:bronze;wood` | 128 | 128 | SC2 | = |  | must |  |
| C7 | AND of options: `materials:bronze, materials:wood` | 0 | 128 | SC3 | ? | Q3.12 | must — fixture with clean names and resources holding one, both, neither |  |
| C8 | Dropdown option, quoted: `"source:Digital Camera"` | 2,224 | 0 | SC2 | = | non-indexed field resolves by name | must |  |
| C9 | One word of a dropdown option: `source:digital` | 0 | 100,862 | SC3 | ? | Q3.12 nothing in core | must |  |
| C10 | Checkbox option, one word: `yspactivity:learning` | 11,671 | 11,671 | SC2 | = |  | must |  |
| C11 | Dynamic keyword option, quoted: `"artistname:Henry Moore"` | 2,713 | 0 | SC2 | = |  | must |  |
| C12 | One word of a dynamic keyword: `artistname:moore` | 2,713 | 100,862 | SC3 | ? | Q3.12 | must — fixture with two options sharing a word |  |
| C13 | Node token: `@@389` | 58,028 | 58,028 | SC1 | = |  | must |  |
| C14 | OR of nodes: `@@389@@388` | 94,936 | 94,936 | SC1 | = |  | must |  |
| C15 | AND of nodes: `@@425, @@389` | 49 | 49 | SC1 | = |  | must |  |
| C16 | AND of nodes, space separated: `@@389 @@425` | 49 | 49 | SC1 | = |  | must |  |
| C17 | NOT a node: `@@!432` | 100,461 | 100,461 | SC1 | = |  | must |  |
| C18 | NOT inside an OR word: `@@!389@@388` | 94,936 | 94,936 | SC1 | = | NOT ignored inside an OR word | must |  |
| C19 | Unknown node: `@@999999999` | 0 | 0 | SC1 | = | unknown node → nothing | must |  |
| C20 | Simple-search-bar checkbox: `@@17913` | 264 | 264 | SC1 | = |  | must |  |
| C21 | Dynamic keyword node: `@@2997` | 33,333 | 33,333 | SC1 | = |  | must |  |
| C22 | GPT keyword node: `@@460` | 37,603 | 37,603 | SC1 | = |  | must |  |
| C23 | Option name and a node of the same field: `orientation:landscape, @@388` | 0 | 0 | SC2 | = |  | must — fixture confirming node 388's field |  |
| C24 | Option names as the search page converts them: `materials:bronze;wood` | 128 | 128 | SC2 | = | the page rewrites it to @@ tokens | must |  |
| C25 | Documented OR shape with a multi-word value: `usagerights:social media;website` | 0 | 0 | SA7 | ↩ | bare OR → suggestion in core | must |  |
| C26 | OR of an option name and a word: `usagerights:website;social` | 82 | 82 | SC2 | = | words discarded by core | must |  |

## D — Dates

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| D1 | Year: `date:2024` | 8,838 | 8,838 | SD1 | = |  | must |  |
| D2 | Year and month: `date:2024-05` | 572 | 572 | SD1 | = |  | must |  |
| D3 | Full date: `date:2024-05-31` | 317 | 317 | SD1 | = |  | must |  |
| D4 | Date stored with a time: `date:2017-05-10` | 626 | 626 | SD1 | = | time of day ignored | must — fixture with a timed value |  |
| D5 | Range: `date:rangestart2024-01-01end2024-02-29` | 822 | 854 | SD3 | = | Q3.4 (a) | must — Q3.4 (a) reproduces it; fixture with a -00 partial and a timed end-day value |  |
| D6 | Range, start only: `date:rangestart2026-01-01` | 2,006 | 2,006 | SD3 | = | Q3.4 (a): start bound only | must |  |
| D7 | Range, end only: `date:rangeend2005-12-31` | 722 | 652 | SD3 | = | Q3.4 (a) | must — plus a row for the stale-bound quirk (date:rangestart…, date:rangeend…) |  |
| D8 | Range as the EDTF box sends it: `date:rangestart2024-00-00end2024-12-99` | 8,838 | 21,151 | SD3 | = | Q3.4 (a) EDTF -00 / -99 | must — fixture with a 2024-12-31 23:xx value |  |
| D9 | Range, end month without a day: `date:rangestart2024-01-01end2024-02-31` | 822 | 21,151 | SD3 | = | Q3.4 (a) end padded -31 | must |  |
| D10 | Simple search year: `basicyear:2024` | 8,838 | 0 | SD2 | ? | Q3.3 / Q3.13 | must |  |
| D11 | Simple search month, any year: `basicmonth:05` | 9,511 | 0 | SD2 | ? | Q3.3 / Q3.13 | must |  |
| D12 | Simple search day, any month: `basicday:05` | 3,227 | 0 | SD2 | ? | Q3.3 / Q3.13 | must |  |
| D13 | Year and month dropdowns: `basicyear:2025, basicmonth:06` | 1,542 | 0 | SD2 | ? | Q3.3 / Q3.13 | must |  |
| D14 | Year, month and day dropdowns: `basicyear:2025, basicmonth:06, basicday:05` | 256 | 0 | SD2 | ? | Q3.3 / Q3.13 | must |  |
| D15 | Legacy "any year" syntax: `date:nnnn\|05` | 9,511 | 0 | SD2 | ? | Q3.3 legacy form | must |  |
| D16 | startdate / enddate form inputs: `startdate:2024-01-01, enddate:2024-12-31` | 0 | 0 | SD2 | ↩ | dead inputs; declined | no — dead inputs; core answers before the hook |  |
| D17 | Date-and-time field, full date: `expirydate:2029-05-31` | 224 | 224 | SD1 | = |  | must |  |
| D18 | Date-and-time field, year: `expirydate:2029` | 3,171 | 3,171 | SD1 | = |  | must |  |
| D19 | Date-and-time field, range: `expirydate:rangestart2029-01-01end2029-12-31` | 3,171 | 3,171 | SD3 | = | Q3.4 (a): the key carries the time of day | must |  |
| D20 | Partial value (year only): `dateofartwork:2025` | 2,593 | 2,593 | SD1 | = | year-only value | must |  |
| D21 | Month, including month-only values: `eventdate:2022-04` | 509 | 509 | SD1 | = | month-only value | must |  |
| D22 | Date field not flagged for indexing: `datephototaken:2024` | 3,676 | 0 | SD5 | = | every field has _q (Q1.1 (b), Q1.2) | must — fixture only |  |
| D23 | Same field, full date: `datephototaken:2012-01-26` | 4,549 | 0 | SD5 | = | every field has _q (Q1.1 (b), Q1.2) | must |  |
| D24 | Date-range field, one value: `daterangetest:2020` | 0 | 2 | SD4 | ✗ | Core 2; Q3.5 | bug — Core 2 (was check) |  |
| D25 | Date-range field, range: `daterangetest:rangestart2020-01-01end2021-12-31` | 1 | 2 | SD4 | = | Q3.5 decided 2026-10-30: overlap via an array _k (SD4 schema addition) | must — rule corrected to overlap: some value >= start AND some value <= end; two _k values per resource (SD4 schema addition); fixture: partial overlap, enclosing, outside, open-ended |  |
| D26 | Two values for one date field: `date:2024, date:2025` | 0 | 0 | SD1 | ? | Q3.14 two values ANDed | must |  |
| D27 | OR on a date: `date:2024;2025` | 0 | 0 | SD1 | ↩ | declined — nothing in core | no — no form emits it; declining gives core's empty answer |  |
| D28 | Another date-and-time field: `copyrightexpirationd:2030` | 47 | 47 | SD1 | = |  | must |  |

## E — Numeric field

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| E1 | Range: `numberfield:numrange0\|100` | 9 | 9 | SE1 | = |  | must — worked with the previous code |  |
| E2 | Exact (one bound): `numberfield:numrange42\|` | 2 | 2 | SE1 | = | single bound = exact | must — worked with the previous code |  |
| E3 | Max only: `numberfield:numrange\|100` | 1 | 1 | SE1 | = |  | must — worked with the previous code |  |
| E4 | Negative bounds: `numberfield:numrangeneg50\|neg1` | 2 | 2 | SE1 | = |  | must — worked with the previous code |  |
| E5 | Decimal bounds: `numberfield:numrange3\|3.14` | 1 | 1 | SE1 | = |  | must — worked with the previous code |  |
| E6 | Plain number in the field: `numberfield:42` | 2 | 0 | SE1 | = | plain value is a keyword in the field | must — it is SB1; fixture with 42.5 and 042 |  |
| E7 | numrange on a text field: `title:numrange1\|10` | 2,937 | 0 | SE1 | ↩ | declined — incidental casting | no — the form emits numrange only for field_constraint = 1 fields; incidental casting; declined |  |

## F — Special searches

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| F1 | !last: `!last100` | 100 | 100 | SF1 | = |  | must |  |
| F2 | !last, first rows: `!last5` fetchrows=0,5 | 5 | 5 | SF1 | = | P29 | must |  |
| F3 | !last with no number: `!lastabc` | 1,000 | 0 | SF1 | ? | Q4.3 non-numeric → 1000 | must |  |
| F4 | !last plus a keyword: `!last100 sculpture` | 1,000 | 1,000 | SF1 | ✗ | Q4.3 space → 1000 (P28) | bug — Core 7 (was must) |  |
| F5 | !last plus a node: `!last1000 @@389` | 1 | 1,000 | SF1 | ✗ | Core 1; Q11.3 | bug — Core 1 (was must) |  |
| F6 | !nodownloads: `!nodownloads` | 94,493 | 0 | SF17 | ↩ | not required — !nodownloads | no |  |
| F7 | !duplicates (all): `!duplicates` | 59 | 0 | SF17 | ↩ | not required — !duplicates | no |  |
| F8 | !duplicates of one resource: `!duplicates123007` | 2 | 0 | SF17 | ↩ | not required — !duplicates | no |  |
| F9 | !collection: `!collection3808` | 55 | 55 | SF2 | = |  | must |  |
| F10 | !collection, public: `!collection3618` | 949 | 949 | SF2 | = |  | must |  |
| F11 | !collection, featured: `!collection3706` | 262 | 262 | SF2 | = |  | must |  |
| F12 | !collection, featured but not public: `!collection3125` | 257 | 257 | SF2 | = |  | must |  |
| F13 | !collection, someone else's private: `!collection158` | 0 | 0 | SF2 | = |  | must |  |
| F14 | !collection, own empty collection: `!collection{owncollection}` | 0 | 0 | SF2 | = |  | must |  |
| F15 | !collection, own upload collection: `!collection-{userref}` | 0 | 0 | SF2 | ↩ | upload collection vetoes (P27) — core answers | must |  |
| F16 | !collection plus a keyword: `!collection3808 sculpture` | 55 | 55 | SF2 | = |  | must |  |
| F17 | !collection, comma then keyword: `!collection3808, sculpture` | 55 | 55 | SF2 | = |  | must |  |
| F18 | !collection plus a node: `!collection3808 @@389` | 1 | 1 | SF2 | = |  | must |  |
| F19 | !collection in collection order: `!collection3808` order_by=collection, sort=asc, fetchrows=0,5 | 55 | 55 | SF2 | = |  | must |  |
| F20 | !collection with a resource type: `!collection3808` restypes=5 | 55 | 55 | SF2 | = | restypes ignored | must |  |
| F21 | !collection with a workflow state: `!collection3808` archive=2 | 55 | 55 | SF2 | = | states ignored, z applies | must |  |
| F22 | !related: `!related121409` | 488 | 0 | SF11 | = | related[] enters with SF11 (Q1.6); Q4.1 settled by the marks | must |  |
| F23 | !related plus a keyword: `!related121409 sculpture` | error: HTTP 0 connecti… | 0 | SF11 | = | as F22 with a keyword; core times out live (slow, not wrong) — fixture only | must |  |
| F24 | !relatedpushed: `!relatedpushed121409` | 0 | 0 | SF11 | = | as F22 plus resource_type:=[push types] | must |  |
| F25 | !geo: `!geo53p5bm1p7t53p7bm1p4` | 4,893 | 0 | SF14 | ↩ | not required — Q4.1 needs a geopoint | no |  |
| F26 | !colourkey: `!colourkeyEPNB` | 1,979 | 0 | SF15 | ↩ | not required — Q4.1 needs colour_key | no |  |
| F27 | !colour: `!colourE` | 70,574 | 0 | SF15 | ↩ | not required — Q4.1 | no |  |
| F28 | !rgb: `!rgb:610,391,405` | 500 | 0 | SF17 | ↩ | not required — !rgb | no |  |
| F29 | !nopreview: `!nopreview` | 105 | 0 | SF12 | = | has_image enters with SF12 (Q1.6); Q4.1 settled by the marks | must |  |
| F30 | !resource: `!resource416` | 1 | 1 | SF4 | = |  | must |  |
| F31 | !resource plus a term with digits: `!resource416, date:2024` | 0 | 1 | SF4 | ✗ | Core 3; Q4.3 | bug — Core 3 (was check) |  |
| F32 | !archivepending: `!archivepending` | 0 | 0 | SF7 | ? | Q4.5 nothing for z1 users | must |  |
| F33 | !userpending: `!userpending` | 0 | 0 | SF7 | ? | Q4.5 | must |  |
| F34 | !contributions: `!contributions9` | 41,283 | 41,283 | SF5 | = |  | must |  |
| F35 | !contributions, own: `!contributions{userref}` | 0 | 0 | SF5 | = | P26 replicated | must |  |
| F36 | !images: `!images` | 100,757 | 0 | SF12 | = | has_image enters with SF12 (Q1.6); Q4.1 settled by the marks | must |  |
| F37 | !unused: `!unused` | 6,967 | 0 | SF16 | ? | Q4.7, Q5.7 | must — if memberships of every collection kind are indexed (Q5.7); core counts private, selection, upload, share-upload and request collections as used |  |
| F38 | !list: `!list416:4460:8692` | 1 | 1 | SF3 | = |  | must |  |
| F39 | !listall: `!listall416:4460:8692` | 1 | 1 | SF3 | = |  | must — treated as !list |  |
| F40 | !list plus a keyword: `!list416:4460:8692 sculpture` | 1 | 1 | SF3 | = | not required; served by composition | no |  |
| F41 | !list with commas: `!list416,4460` | 1 | 1 | SF3 | = | not required; served by composition — comma ends the list | no |  |
| F42 | !hasdata: `!hasdata170` | 12 | 12 | SF6 | = |  | must |  |
| F43 | !hasdata on a non-indexed field: `!hasdata93` | 63,316 | 63,316 | SF6 | = |  | must |  |
| F44 | !hasdata plus a keyword: `!hasdata170 sculpture` | 0 | 0 | SF6 | = | not required; served by composition | no |  |
| F45 | !empty by ref: `!empty170` | 100,850 | 0 | SF10 | ? | Q4.8 | must |  |
| F46 | !empty by short name: `!emptynumberfield` | 100,004 | 0 | SF10 | ✗ | Core 5; Q4.8 | bug — Core 5 (was no) |  |
| F47 | !empty on a field of one type: `!empty95` | 100,457 | 0 | SF10 | ✗ | Core 6; Q4.8 | bug — Core 6 (was must) |  |
| F48 | !empty with a resource type: `!empty170` restypes=1 | 93,054 | 0 | SF10 | ? | Q4.8 | must — type filter by construction (field 170 is global) |  |
| F49 | !empty, unknown field: `!emptynosuchfield` | 100,004 | 0 | SF10 | ✗ | Core 5; Q4.8 | bug — Core 5 (was no) |  |
| F50 | !properties file extension: `!propertiesfext:jpg` | 66,810 | 0 | SF9 | = | file_extension enters with SF9 (Q1.6); Q4.1 settled by the marks, Q4.11 decided 2026-10-30 | must |  |
| F51 | !properties, not an extension: `!propertiesfext:-jpg` | 34,052 | 0 | SF9 | = | NOT LIKE on file_extension (SF9) | must |  |
| F52 | !properties, extension wildcard: `!propertiesfext:jp*` | 71,886 | 0 | SF9 | = | Q4.11 decided 2026-10-30: the LIKE as a prefix | must — expectation written as the LIKE |  |
| F53 | !properties minimum height: `!propertieshmin:3000` | 69,894 | 0 | SF9 | = | height from resource_dimensions (SF9) | must |  |
| F54 | !properties, several: `!propertieswmin:4000;hmin:3000` | 55,705 | 0 | SF9 | = | two keys ANDed (SF9) | must |  |
| F55 | !properties minimum size: `!propertiesfmin:5` | 53,138 | 0 | SF9 | = | file_size, MB to bytes (SF9) | must |  |
| F56 | !properties maximum size: `!propertiesfmax:1` | 11,993 | 0 | SF9 | = | file_size, MB to bytes (SF9) | must |  |
| F57 | !properties has preview: `!propertiespi:1` | 100,072 | 0 | SF9 | = | has_image (SF9, SF12) | must |  |
| F58 | !properties contributor: `!propertiescu:9` | 41,283 | 0 | SF9 | = | created_by (SF9) | must |  |
| F59 | !properties orientation: `!propertiesorientation:portrait` | 36,092 | 0 | SF9 | = | Q4.11 decided 2026-10-30: orientation from the dimensions, no dimensions row = square | must — fixture with a dimension-less resource |  |
| F60 | !properties plus a keyword: `!propertiesfext:jpg sculpture` | 32,459 | 0 | SF9 | = | properties then the keyword (SF9) | must |  |
| F61 | !properties plus a node: `!propertiesfext:jpg @@389` | 41,004 | 0 | SF9 | = | properties then the node (SF9) | must |  |
| F62 | !properties, unknown property: `!propertiesbogus:1` | 100,862 | 0 | SF9 | = | Q4.11 decided 2026-10-30: an unknown key adds no filter | must — unknown property names ignored inside a real property search (bogus:1;fext:jpg = fext:jpg) |  |
| F63 | !integrityfail: `!integrityfail` | 0 | 0 | SF13 | ↩ | declined — maintenance flag | no — a maintenance flag set outside the indexing pipeline; declined |  |
| F64 | !locked: `!locked` | 1 | 0 | SF13 | = | lock_user enters with SF13 (Q1.6); Q4.1 settled by the marks | must |  |
| F65 | !noningested (admin only): `!noningested` | 100,862 | 0 | SF13 | ↩ | declined — F67's bug | no — F67's bug (an unknown special returns everything); admin half applies no $sql_filter; declined |  |
| F66 | !report: `!report3p365` | 0 | 0 | SF17 | ↩ | not required — !report | no |  |
| F67 | Unknown special search: `!nosuchsearch` | 100,862 | 0 | SF8 | ✗ | Q4.9 unknown special → everything in core | bug — an unknown special search returns everything (was must) |  |
| F68 | Two special searches: `!last100 !collection3808` | 1,000 | 0 | SF8 | ✗ | first special only; !last1000 | bug — Core 7 (was must) |  |
| F69 | !last, comma before the keyword: `!last100, sculpture` | 100 | 100 | SF1 | = | comma keeps 100 | must — live: refs compared with F1 |  |

## G — Parameters

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| G1 | One resource type: `sculpture` restypes=1 | 45,727 | 45,727 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G2 | Several resource types: `sculpture` restypes=1,5 | 48,620 | 48,620 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G3 | Resource types "Global": `sculpture` restypes=Global | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = | Global ignored | must |  |
| G4 | "Global" followed by a type: `sculpture` restypes=Global,5 | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G5 | Workflow state 0: `sculpture` archive=0 | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G6 | Two states: `sculpture` archive=0,8 | 52,848 | 52,848 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G7 | Custom state only: `sculpture` archive=8 | 3,380 | 3,380 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G8 | State hidden by a z permission: `sculpture` archive=2 | 0 | 0 | [D5](../D-search-pipeline.md) | = | z exclusion with $uploader_view_override | must — D5's z exclusion includes the uploader override; fixture with an own state-2 resource |  |
| G9 | Pending submission: `sculpture` archive=-1 | 0 | 0 | [D5](../D-search-pipeline.md) | = | pending rule | must — fixture with own and others' -1 resources |  |
| G10 | Empty archive: `sculpture` archive= | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = | default states | must |  |
| G11 | Order: relevance (default): `sculpture` order_by=relevance, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | ≈ | Q7.1 (B): keyword relevance differs, keyword-less exact | must — accepted difference under Q7.1: keyword searches use the text-match score in place of core's node hit-count score; keyword-less relevance exact |  |
| G12 | Order: popularity: `sculpture` order_by=popularity, fetchrows=0,5 | 49,468 | 0 | SG2 | = | packed key ur_hits; fresh via E27 | must — composite key (user_rating, hit_count) plus date and ref; exact as of the last hit-count copy |  |
| G13 | Order: rating: `sculpture` order_by=rating, fetchrows=0,5 | 49,468 | 0 | SG2 | ≈ | rating_ur key; keyword tie-break differs (Q7.1) | must — accepted difference for keyword searches: after r.rating and user_rating the text-match score replaces core's hit-count tie-breaker; documented and tested apart from G11 |  |
| G14 | Order: date: `sculpture` order_by=date, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | date | must |  |
| G15 | Order: colour: `sculpture` order_by=colour, fetchrows=0,5 | 49,468 | 0 | SG3 | ↩ | not required — Q7.2 six keys | no |  |
| G16 | Order: title: `sculpture` order_by=title, fetchrows=0,5 | 49,468 | 0 | SG1 | = | title sort key; collation checked live (H4) | must — live collation check |  |
| G17 | Order: file_path: `sculpture` order_by=file_path, fetchrows=0,5 | 49,468 | 0 | SG1 | ↩ | declined — admin-only key | no — unexercised admin-only key; core answers |  |
| G18 | Order: resourceid: `sculpture` order_by=resourceid, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = |  | must |  |
| G19 | Order: resourcetype: `sculpture` order_by=resourcetype, fetchrows=0,5 | 49,468 | 0 | SG1 | = | type order key + sync (SG1 schema addition, E24) | must — index the type's configured order and keep it synchronised; sort by order, type, ref with missing_values first (asc) / last (desc) |  |
| G20 | Order: extension: `sculpture` order_by=extension, fetchrows=0,5 | 49,468 | 0 | SG1 | = | Q7.2 | must |  |
| G21 | Order: status: `sculpture` order_by=status, fetchrows=0,5 | 49,468 | 0 | SG1 | = | Q7.2 | must — rerun with archive=0,8 |  |
| G22 | Order: modified: `sculpture` order_by=modified, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | modified, ref | must |  |
| G23 | Order: random: `sculpture` order_by=random, fetchrows=0,5 | 49,468 | 0 | SG5 | ↩ | not required — random never | no |  |
| G24 | Order: country: `sculpture` order_by=country, fetchrows=0,5 | 49,468 | 0 | SG1 | = | not required; served by composition — falls back to relevance when field 3 is absent (= G11) | no |  |
| G25 | Order: a field by number (date): `sculpture` order_by=field12, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | field12 is the date field | must — field 12 is the date field |  |
| G26 | Order: numeric field: `numberfield:numrangeneg100\|2000` order_by=field170, sort=asc, fetchrows=0,12 | 0 | 0 | SG4 | ✗ | Core 8; Q7.9 | bug — Core 8 (was check); core returns nothing here (Core 8); satisfied by declining |  |
| G27 | Order: title field by number: `sculpture` order_by=field8, fetchrows=0,5 | 49,468 | 0 | SG4 | = | Q7.9 decided 2026-10-30: a text sort key for the title field; G26 and G43 stay bug rows | must |  |
| G28 | Order: unknown: `sculpture` order_by=bogus, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | unknown → relevance (= G11) | must |  |
| G29 | Sort ascending: `sculpture` order_by=resourceid, sort=asc, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = |  | must |  |
| G30 | Sort: invalid value: `sculpture` order_by=resourceid, sort=sideways, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | invalid sort → asc | must |  |
| G31 | Window: offset and count: `sculpture` order_by=resourceid, sort=asc, fetchrows=0,5 | 49,468 | 49,468 | [D10](../D-search-pipeline.md) | = |  | must |  |
| G32 | Window: count only (legacy): `sculpture` order_by=resourceid, sort=asc, fetchrows=5 | 5 rows | 5 rows | [D10](../D-search-pipeline.md) | = |  | must |  |
| G33 | Window: legacy with an offset: `sculpture` order_by=resourceid, sort=asc, fetchrows=5, offset=10 | 5 rows | 5 rows | [D10](../D-search-pipeline.md) | = |  | must |  |
| G34 | Recent days: 365: `sculpture` daylimit=365 | 10,365 | 10,365 | [D5](../D-search-pipeline.md) | = | day limit | must |  |
| G35 | Recent days: 90: (empty) daylimit=90 | 3,089 | 3,089 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G36 | Recent days: 30: (empty) daylimit=30 | 0 | 0 | [D5](../D-search-pipeline.md) | = |  | must |  |
| G37 | go (previous / next): `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | · | go: logging only; local harness | n/a — logging only |  |
| G38 | editable_only: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | ↩ | editable_only deferred (Q6.4); local harness | no — deferred under Q6.4 until sync and read-time validation work; declined, core's permission filtering answers; the stale-variable defect is Q11.3's |  |
| G39 | access: `(any)` | n/a | n/a | [D6](../D-search-pipeline.md) | = | P25; local harness | must — index the numeric access level; filter only when the user has v and the argument is numeric; frontend fixtures (the API does not expose it) |  |
| G40 | smartsearch: `(any)` | n/a | n/a | [D5](../D-search-pipeline.md) | · | D5 reads $archive_standard; local harness | n/a — via the API; D5 must read $archive_standard when reproducing search_filter() |  |
| G41 | ignore_filters: `(any)` | n/a | n/a | [D2](../D-search-pipeline.md) | · | dead flag; local harness | n/a — dead |  |
| G42.1 | return_refs_only: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | = | refs-only shapes; local harness | must — preserve the fetchrows-dependent shapes; specials omit ORDER BY for refs-only, so compare membership without order |  |
| G42.2 | return_disk_usage: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | ↩ | disk usage declines; local harness | no — declined to core for its SQL aggregation wrapper |  |
| G42.3 | returnsql: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | ↩ | returnsql declines; local harness | no — declined so SQL callers receive the query object |  |
| G42.4 | access_override: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | = | access_override (D6/D7); local harness | must — by construction: override of custom-access joins, T restrictions, confidential filtering, the archive/pending block and z exclusions |  |
| G43 | Order: a field that is not a join column: `sculpture` order_by=field170, fetchrows=0,5 | 0 | 0 | SG4 | ✗ | Core 8; Q7.9 | bug — Core 8 (was must); core returns nothing (Core 8); satisfied by declining |  |

## H — Combinations

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| H1 | Keyword and node: `sculpture, @@389` | 26,893 | 26,893 | SC1 | = |  | must |  |
| H2 | Keyword and option name: `sculpture, orientation:landscape` | 26,893 | 26,893 | SC2 | = |  | must |  |
| H3 | Field term and node: `title:sculpture, @@389` | 2,439 | 2,439 | SB1 | = |  | must |  |
| H4 | Keyword and two nodes: `sculpture, @@425, @@389` | 41 | 41 | SC1 | = |  | must |  |
| H5 | Field wildcard and node: `title:sculpt*, @@389` | 2,507 | 2,507 | SB3 | = | as B6, ANDed with the node | must |  |
| H6 | Node and NOT node: `@@425, @@!389` | 99 | 99 | SC1 | = |  | must |  |
| H7 | Keyword and date: `sculpture, date:2024` | 4,517 | 4,517 | SD1 | = |  | must |  |
| H8 | Keyword, node and date, by date: `sculpture, @@389, date:2024` order_by=date, fetchrows=0,5 | 2,034 | 2,034 | SD1 | = | order by D9 | must |  |
| H9 | Keyword, type, by ref: `sculpture` restypes=5, order_by=resourceid, sort=asc, fetchrows=0,5 | 2,893 | 2,893 | [D5](../D-search-pipeline.md) | = |  | must |  |
| H10 | Keyword, node, type and two states: `sculpture, @@389` restypes=5, archive=0,2 | 1,408 | 1,408 | [D5](../D-search-pipeline.md) | = |  | must |  |
| H11 | Collection and keyword, collection order: `!collection3808 sculpture` order_by=collection, sort=asc, fetchrows=0,5 | 55 | 55 | SF2 | = |  | must |  |
| H12 | Last 100 and keyword: `!last100 sculpture` fetchrows=0,5 | 1,000 | 1,000 | SF1 | ✗ |  | bug — Core 7, the same string as F4: core reads !last1000; Chris's mark: must |  |
| H13 | Node and NOT keyword: `@@425, -sculpture` | 48 | 48 | SA4 | = |  | must |  |
| H14 | Phrase and node: `"sculpture park", @@389` | 3,412 | 0 | SA3 | ? | Q2.5 | must |  |
| H15 | Keyword and NOT keyword: `sculpture, -landscape` | 15,049 | 15,059 | SA4 | = |  | must |  |
| H16 | Keyword and numeric range: `sculpture, numberfield:numrange0\|100` | 0 | 0 | SE1 | = |  | must — fixture with a resource holding both |  |
| H17 | Node and date: `@@389, date:2024` | 4,169 | 4,169 | SD1 | = |  | must |  |
| H18 | OR of options and a node: `materials:bronze;wood, @@389` | 18 | 18 | SC2 | = |  | must |  |
| H19 | Simple-search year and keyword: `basicyear:2024, sculpture` | 4,517 | 0 | SD2 | ? | Q3.3 | must |  |
| H20 | Simple-search year and node: `basicyear:2024, @@389` | 4,169 | 0 | SD2 | ? | Q3.3 | must |  |
| H21 | Two date fields: `date:2024, expirydate:2029` | 2,972 | 2,972 | SD1 | = |  | must |  |
| H22 | Two wildcards: `sculpt*, land*` | 36,026 | 0 | SA5 | = | two prefix terms ANDed; parity proved by SA5's battery | must — each term a prefix over every eligible visible field; tokenisation parity and candidate caps (max_filter_by_candidates) proved by SA5's battery |  |
| H23 | Phrase and NOT phrase: `"sculpture park", -"black and white"` | 6,758 | 0 | SA4 | ↩ | not required — Q2.6 negated phrase (SA3 first) | no |  |
| H24 | Four keywords: `sculpture landscape nature outdoor` | 19,608 | 19,608 | SA1 | = |  | must |  |
| H25 | Two fields, node and date: `title:sculpture, credit:wilde, @@389, date:2024` | 65 | 65 | SB1 | = |  | must |  |
| H26 | Collection and date: `!collection3808 date:2025` | 55 | 55 | SF2 | = |  | must |  |
| H27 | Contributions, keyword and node: `!contributions9 sculpture, @@389` | 9,240 | 9,240 | SF5 | = |  | must |  |
| H28 | List and node: `!list416:4460:8692 @@389` | 1 | 1 | SF3 | = |  | must |  |
| H29 | Hasdata and node: `!hasdata170 @@389` | 5 | 5 | SF6 | = |  | must |  |
| H30 | Properties and date: `!propertiesfext:jpg date:2024` | 6,224 | 0 | SF9 | = | properties then the date (SF9) | must |  |
| H31 | Everything at once: `sculpture, "sculpture park", @@389, orientation:landscape, date:2024, -"black and white"` restypes=1, archive=0, order_by=date, sort=desc, fetchrows=0,5 | 307 | 0 | SH1 | = | capstone; needs SA3 SA4 SD1 SC2 D5 D9 | must — the fixture proves the NOT phrase and the adjacency separately; expectation under "H31, the capstone" in checks-findings.md |  |
| H32 | Bare OR and a node: `sculpture;landscape, @@389` | 58,028 | 0 | SA7 | ✗ |  | bug — Core 9; Chris's mark: check |  |
| H33 | Wildcard OR and a node: `sculpt*;land*, @@389` | 58,028 | 0 | SA7 | ↩ | not required | no |  |
| H34 | Field OR and a node: `title:sculpture;park, @@389` | 2,653 | 0 | SA7 | = | Q2.7 mechanism 4: the field OR as one filter clause, ANDed with the node | must |  |
| H35 | Date range, node and keyword: `date:rangestart2024-01-01end2024-12-31, @@389, sculpture` | 2,030 | 2,034 | SD3 | = | Q3.4 (a), ANDed with the node and the keyword | must |  |
| H36 | Year as a word and a keyword: `2024, sculpture` | 5,477 | 5,476 | SA9 | ? | Q2.11 | must |  |
| H37 | Type name and keyword: `document sculpture` | 111 | 101 | SA9 | ? | Q2.9 | must |  |
| H38 | Resource types and collection ignored together: `!collection3618 sculpture` restypes=5, archive=8 | 409 | 409 | SF2 | = |  | must — the parameters are ignored, z still applies |  |

## I — Syntax edge cases

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| I1 | No space after a special search: `!collection3808,sculpture` | 55 | 55 | [D2](../D-search-pipeline.md) | = | not required; served by composition — keywords only after the first space | no |  |
| I2 | Comma without a space: `sculpture,landscape` | 22 | 34,409 | SA6 | ✗ | Q2.8 phrase in core | bug — Core 10; Chris's mark: check |  |
| I3 | Comma inside a field term: `title:sculpture,landscape` | 0 | 25 | SB4 | ✗ | Core 10; Q11.3 | bug — Core 10, the field form of I2; was check; same disposition as I2 |  |
| I4 | Repeated word: `sculpture sculpture` | 49,468 | 49,468 | [D2](../D-search-pipeline.md) | = |  | must |  |
| I5 | Surrounding spaces: `   sculpture   ` | 49,468 | 49,468 | [D2](../D-search-pipeline.md) | = |  | must |  |
| I6 | Trailing colon: `sculpture:` | 49,468 | 49,468 | SB1 | = | unknown name → the colon becomes a space | must — deterministic |  |
| I7 | Leading colon: `:sculpture` | 49,468 | 49,468 | SB1 | = | empty name → same as I6 | must — the plugin's field lookup is literal like core's |  |
| I8 | Unbalanced quote: `"sculpture` | 49,468 | 0 | SA3 | ↩ | declined — accidental path | no — an accidental path; declined |  |
| I9 | Upper-case special search: `!LAST100` | 100,862 | 1,000 | SF8 | ✗ | Q4.9 case-sensitive → unknown | bug — an unknown special search returns everything (F67); Chris's mark: check |  |
| I10 | Nodes without a space: `@@389,@@425` | 49 | 49 | SC1 | = |  | must |  |
| I11 | Upper-case field name and value: `Title:Sculpture` | 4,226 | 4,226 | SB1 | = |  | must |  |
| I12 | Dropdown value from the advanced form: `"source:igital Camer"` | 0 | 0 | SJ1 | ↩ | unreachable; declined | no — unreachable branch |  |

## J — Strings as the advanced search page assembles them

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| J1 | Resource IDs, words, a field, nodes, type and state: `!list2181:3879:4093 sculpture, landscape, title:sculpture, @@389@@388` restypes=5, archive=0 | 1 | 1 | SJ1 | = |  | must |  |
| J2 | The same with a property filled in: `!propertiesfext:jpg ,!list2181:3879:4093 sculpture, landscape, title:sculpture, @@389@@388` restypes=5, archive=0 | 8 | 0 | SJ1 | ✗ | Q4.10 | bug — the !properties and !list prefixes collide; Chris's mark: must |  |
| J3 | Posted special search prefix: `!contributions9,sculpture` | 41,283 | 41,283 | SJ1 | = | I1 rule | must |  |
| J4 | Posted special search prefix, more terms: `!contributions9,title:park, sculpture` | 19,396 | 19,396 | SJ1 | = |  | must — the ref parse takes the leading integer |  |
| J5 | Range with a start year only: `date:rangestart2024` | 21,151 | 21,151 | SD3 | = | Q3.4 (a): the unpadded year keys as 20240000000000 | must |  |
| J6 | Range with months only: `date:rangestart2024-05end2024-06-31` | 1,947 | 19,079 | SD3 | = | Q3.4 (a): month-only start; the end padded as core pads it | must |  |
| J7 | Live result count: `sculpture` fetchrows=0,0 | 49,468 | 49,468 | [D10](../D-search-pipeline.md) | = | count only | must |  |
| J8 | Checkbox list, two options ticked: `sculpture, @@6125@@6130` | 284 | 284 | SC2 | = |  | must |  |
| J9 | Checkbox list with $checkbox_and: `sculpture, @@6125, @@6130` | 0 | 0 | SC2 | = |  | must — live: each node alone > 0 |  |
| J10 | Dropdown choice: `sculpture, @@247` | 1,429 | 1,429 | SC2 | = |  | must |  |
| J11 | Property filter with terms: `!propertiesfext:jpg ,sculpture, title:park` | 547 | 0 | SJ1 | = | properties then the terms (SF9); Q4.1 settled by the marks | must |  |
| J12 | Full-text box with a word: `sculpture, "@FULL_TEXT:distant island"` | 461 | 0 | SA8 | = | Q2.15: the box appended to a word | must |  |
| J13 | Date value and range on one field: `date:2024-05, date:rangestart2024-05-01end2024-05-31` | 524 | 572 | SD3 | = | a value (SD1) and a range on one field, ANDed | must — three joins ANDed; fixture with a 2024-05-00 value |  |

## K — What the indexing flags control

| Row | Search | Core | Old | Owner | Plan | Note | Required | Done |
|---|---|---|---|---|---|---|---|---|
| K1 | Word that exists only in a non-indexed field: `nikon` | 0 | 0 | SK1 | · | core answers before the hook | must |  |
| K2 | The same word with a wildcard: `nikon*` | 23,412 | 0 | SK1 | = | Q1.1 (b): the values are in the index | must — fixture must not assert the suggestion text after a wildcard run |  |
| K3 | Wildcard inside the non-indexed field: `camera:nik*` | 23,412 | 0 | SK1 | = | Q1.1 (b): prefix on the field's representation | must — fixture: nik* matches, ni* and nikon do not |  |
| K4 | Full-text box on a non-indexed field: `"@FULL_TEXT:nikon"` | 23,412 | 0 | SA8 | = | Q1.1 (b): the box queries every visible field | must |  |
| K5 | Bare fragment of a partially indexed field: `ksho` | 665 | 665 | SB6 | = | Q1.3 (a): _p in query_by | must |  |
| K6 | Word in an inactive field: `description:sculpture` | 0 | 57 | SK2 | = | P37 | must |  |
| K7 | Phrase held only in non-indexed fields: `"library book"` | 0 | 0 | SK1 | · | core answers before the hook | must |  |
| K8 | The same phrase through full text: `"@FULL_TEXT:[QUOTES]library book[QUOTES]"` | 1 | 0 | SA8 | = | Q1.1 (b): the box queries every visible field | must — fixture with the phrase on two resources |  |
| K9 | Complete indexing: `field:two words` | n/a | n/a | SK3 | = | whole-value key; SK3 schema addition | must — whole-value keyword (_c, SK3 schema addition); no such field on the test system |  |
| K10 | Word beyond the 500th character: `field:word` | n/a | n/a | SK3 | = | uncapped _w; SK3 schema addition | must — fixture: unique word at character 600: word 0, wor* 1, *ord 0, full-text box 1 (uncapped _w, SK3 schema addition) |  |
| K11 | Number range on a non-indexed numeric field: `field:numrange1\|10` | n/a | n/a | SE1 | = | every field has _q (Q1.1 (b)) | must — fixture only |  |
| K12 | Field hidden from the user by permission: `field:word` | n/a | n/a | SK2 | = | core returns false before the hook; local harness | must — core returns false before the hook |  |

## Totals

Re-derived from the catalogue marks on 2026-10-20 and again on 2026-10-30 when the 77 *check* rows were
decided (G42 split into four): *bug* rows are `✗` (Q11.3 decides each), *no* rows are `↩` unless the
mechanism that serves a required row serves them at no cost (then `=`, noted as "served by
composition"), *n/a* rows are `·`, *must* rows are `=` where the decisions settle the mechanism, `≈`
for the two accepted differences under Q7.1 (G11, G13), `?` where a question is still open (`↩` only
where declining is how core's own answer is reached). No *check* rows remain. On the same day the `?` rows were swept against the decisions log: the 41 rows whose named questions were already settled (Q1.1, Q1.3, Q1.6, Q2.7, Q2.15, Q3.4, Q3.5, Q4.11, Q7.9 and the 2026-10-30 marks) moved to `=` with the decision in the note; the 43 that remain each name a question that is still open.

| Plan | Rows | must | bug | no | n/a |
|---|---|---|---|---|---|
| = | 207 | 202 | 0 | 5 | 0 |
| ≈ | 2 | 2 | 0 | 0 | 0 |
| ? | 43 | 43 | 0 | 0 | 0 |
| ↩ | 29 | 2 | 0 | 27 | 0 |
| ✗ | 19 | 0 | 19 | 0 | 0 |
| · | 6 | 3 | 0 | 0 | 3 |

## Package standing

A package is **required** when it owns at least one *must* row, **bug only** when every row is a core
bug, **parked** when every row is *no* or *n/a*. No package is pending check since 2026-10-30. The pipeline
tasks (D1–D10) are required whatever their rows say. The former check rows are gathered in
[checks.md](checks.md) with their findings and decisions.

| Package | Standing | must | bug | no | n/a |
|---|---|---|---|---|---|
| D1 | pipeline | 2 | 0 | 3 | 1 |
| D2 | pipeline | 2 | 0 | 1 | 1 |
| D5 | pipeline | 15 | 0 | 0 | 1 |
| D6 | pipeline | 1 | 0 | 0 | 0 |
| D9 | pipeline | 8 | 0 | 0 | 0 |
| D10 | pipeline | 4 | 0 | 0 | 0 |
| SA1 | required | 7 | 0 | 0 | 0 |
| SA2 | required | 2 | 0 | 0 | 0 |
| SA3 | required | 4 | 0 | 1 | 0 |
| SA4 | required | 4 | 0 | 1 | 0 |
| SA5 | required | 4 | 0 | 1 | 0 |
| SA6 | required | 2 | 1 | 0 | 0 |
| SA7 | required | 4 | 2 | 1 | 0 |
| SA8 | required | 7 | 0 | 0 | 0 |
| SA9 | required | 6 | 0 | 0 | 0 |
| SA10 | required | 2 | 0 | 0 | 0 |
| SA11 | required | 1 | 0 | 1 | 0 |
| SA12 | required | 1 | 0 | 0 | 0 |
| SB1 | required | 10 | 0 | 0 | 0 |
| SB2 | required | 2 | 0 | 1 | 0 |
| SB3 | required | 3 | 0 | 0 | 0 |
| SB4 | required | 2 | 1 | 0 | 0 |
| SB5 | required | 1 | 1 | 1 | 0 |
| SB6 | required | 2 | 0 | 0 | 0 |
| SC1 | required | 14 | 0 | 0 | 0 |
| SC2 | required | 14 | 0 | 0 | 0 |
| SC3 | required | 7 | 0 | 0 | 0 |
| SD1 | required | 14 | 0 | 1 | 0 |
| SD2 | required | 8 | 0 | 1 | 0 |
| SD3 | required | 10 | 0 | 0 | 0 |
| SD4 | required | 1 | 1 | 0 | 0 |
| SD5 | required | 2 | 0 | 0 | 0 |
| SE1 | required | 8 | 0 | 1 | 0 |
| SF1 | required | 4 | 3 | 0 | 0 |
| SF2 | required | 16 | 0 | 0 | 0 |
| SF3 | required | 3 | 0 | 2 | 0 |
| SF4 | required | 1 | 1 | 0 | 0 |
| SF5 | required | 3 | 0 | 0 | 0 |
| SF6 | required | 3 | 0 | 1 | 0 |
| SF7 | required | 2 | 0 | 0 | 0 |
| SF8 | bug only | 0 | 3 | 0 | 0 |
| SF9 | required | 14 | 0 | 0 | 0 |
| SF10 | required | 2 | 3 | 0 | 0 |
| SF11 | required | 3 | 0 | 0 | 0 |
| SF12 | required | 2 | 0 | 0 | 0 |
| SF13 | required | 1 | 0 | 2 | 0 |
| SF14 | parked | 0 | 0 | 1 | 0 |
| SF15 | parked | 0 | 0 | 2 | 0 |
| SF16 | required | 1 | 0 | 0 | 0 |
| SF17 | parked | 0 | 0 | 5 | 0 |
| SG1 | required | 4 | 0 | 2 | 0 |
| SG2 | required | 2 | 0 | 0 | 0 |
| SG3 | parked | 0 | 0 | 1 | 0 |
| SG4 | required | 1 | 2 | 0 | 0 |
| SG5 | parked | 0 | 0 | 1 | 0 |
| SH1 | required | 1 | 0 | 0 | 0 |
| SJ1 | required | 4 | 1 | 1 | 0 |
| SK1 | required | 7 | 0 | 0 | 0 |
| SK2 | required | 2 | 0 | 0 | 0 |
| SK3 | required | 2 | 0 | 0 | 0 |

Rows per owner: D1 6 · D2 4 · D5 16 · D6 1 · D9 8 · D10 4 · SA1 7 · SA2 2 · SA3 5 · SA4 5 · SA5 5 · SA6 3 · SA7 7 · SA8 7 · SA9 6 · SA10 2 · SA11 2 · SA12 1 · SB1 10 · SB2 3 · SB3 3 · SB4 3 · SB5 3 · SB6 2 · SC1 14 · SC2 14 · SC3 7 · SD1 15 · SD2 9 · SD3 10 · SD4 2 · SD5 2 · SE1 9 · SF1 7 · SF2 16 · SF3 5 · SF4 2 · SF5 3 · SF6 4 · SF7 2 · SF8 3 · SF9 14 · SF10 5 · SF11 3 · SF12 2 · SF13 3 · SF14 1 · SF15 2 · SF16 1 · SF17 5 · SG1 6 · SG2 2 · SG3 1 · SG4 3 · SG5 1 · SH1 1 · SJ1 6 · SK1 7 · SK2 2 · SK3 2
