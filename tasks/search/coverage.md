# Catalogue coverage

Every row of the core search catalogue (`docs/core-search-catalogue.md` on `typesense_search_catalogue`, measured 6–9 October 2026) with the package that owns it and what the plan intends for it. The package files are in this folder; the pipeline tasks (D1–D11) are in [../D-search-pipeline.md](../D-search-pipeline.md). **Core** is the live total for the core user; **Old** is what the previous plugin returned (0 usually means it declined in Typesense-only mode). A row is owned by exactly one package; it may be a case in others.

Plan: `=` serve with core's result · `↩` decline, core answers · `≈` accepted difference (needs a decision row) · `✗` a core bug, disposition per Q11.3 · `·` nothing to build (core answers before the hook, or the form is unreachable) · `?` depends on the question named in the note. Rows marked *local harness* cannot be sent through the API and are covered by the local batteries only (Q10.14).

Status column: blank until the package's battery is green for the row.


## A — Keywords

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| A1 | One keyword: `sculpture` | 49,468 | 49,468 | SA1 | = |  |  |
| A2 | Two keywords: `sculpture landscape` | 34,408 | 34,409 | SA1 | = |  |  |
| A3 | Comma separated: `sculpture, landscape` | 34,408 | 34,409 | SA1 | = |  |  |
| A4 | Upper case: `SCULPTURE Park` | 19,680 | 19,688 | SA1 | = |  |  |
| A5 | Plural (stemming): `sculptures` | 49,468 | 49,468 | SA12 | = | stemmer drift Q2.14 |  |
| A6 | Quoted single word: `"sculpture"` | 49,069 | 0 | SA3 | ? | Q2.5 |  |
| A7 | Quoted phrase: `"sculpture park"` | 6,883 | 0 | SA3 | ? | Q2.5 |  |
| A8 | Phrase with a stop word: `"black and white"` | 3,179 | 3,179 | SA3 | ? | Q2.4 stop-word gap |  |
| A9 | NOT a keyword: `-landscape` | 32,045 | 32,078 | SA4 | = | the 33 extra are inactive-field keywords (SK2, Q6.7) |  |
| A10 | NOT a phrase: `-"sculpture park"` | 93,979 | 100,862 | SA4 | ? | Q2.6 |  |
| A11 | Trailing wildcard: `sculpt*` | 51,007 | 49,610 | SA5 | ? | Q2.3; full parity needs non-indexed values (Q1.1) |  |
| A12 | Leading wildcard: `*scape` | 68,601 | 381 | SA5 | ? | Q2.3 leading wildcard |  |
| A13 | Wildcard inside a word: `sc*pture` | 69,679 | 0 | SA5 | ? | Q2.17 behaves as sc* in core |  |
| A14 | Short wildcard (under 3 letters): `so*` | 15,618 | 6,676 | SA5 | ? | Q2.16 keyword-table branch |  |
| A15 | OR of two wildcards: `land*;sun*` | 71,530 | 0 | SA7 | ↩ | wildcard OR |  |
| A16 | OR of two plain words: `sculpture;landscape` | 83,840 | 0 | SA7 | ↩ | Core 9 (suggestion first, OR later) |  |
| A17 | Stop word alone: `the` | 100,862 | 32,537 | SA2 | ? | Q2.4 stop word only |  |
| A18 | Stop word plus a word: `the sculpture` | 49,468 | 17,228 | SA2 | = | after stripping $noadd |  |
| A19 | Hyphenated word: `black-and-white` | 3,179 | 3,194 | SA6 | ? | Q2.8 |  |
| A20 | Apostrophe: `atkinson's` | 2 | 3 | SA6 | ? | Q2.8 |  |
| A21 | Related keyword: `antony` | 1,491 | 205 | SA10 | ? | Q2.10 |  |
| A22 | The related word itself: `anthony` | 1,491 | 1,300 | SA10 | ? | Q2.10 |  |
| A23 | Resource type name: `document` | 490 | 375 | SA9 | ? | Q2.9 / Q1.7 |  |
| A24 | Contributor name: `admin` | 13 | 16 | SA9 | ? | Q2.9 / Q1.7; option off on the test system |  |
| A25 | A resource number: `416` | 100 | 0 | SA9 | ? | Q7.4 exact ref first; Q2.11 |  |
| A26 | A year as a word: `2024` | 10,289 | 0 | SA9 | ? | Q2.11 year as a word |  |
| A27 | Full-text box: `"@FULL_TEXT:distant island"` | 956 | 0 | SA8 | ↩ | full-text box |  |
| A28 | Full-text with operators: `"@FULL_TEXT:+distant +island"` | 9 | 0 | SA8 | ↩ |  |  |
| A29 | Full-text phrase: `"@FULL_TEXT:[QUOTES]distant island[QUOTES]"` | 8 | 0 | SA8 | ↩ |  |  |
| A30 | Misspelt word: `sculptre` | 1 | 1 | SA11 | · | core answers before the hook |  |
| A31 | Known plus unknown word: `sculpture zzzzqqq` | 0 | 0 | SA11 | · | core answers before the hook |  |
| A32 | Only an asterisk: `*` | 100,862 | 100,862 | SA1 | = | empty search |  |
| A33 | Empty search: (empty) order_by=resourceid | 100,862 | 100,862 | SA1 | = | order by D9 |  |
| A34 | Full-text exclusion: `"@FULL_TEXT:+distant -island"` | 880 | 880 | SA8 | ↩ |  |  |

## B — Field-specific terms (text fields)

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| B1 | Word in a field: `title:sculpture` | 4,226 | 4,226 | SB1 | = |  |  |
| B2 | Two words after a field: `title:henry moore` | 251 | 251 | SB2 | = | by construction |  |
| B3 | Quoted phrase in a field: `"title:henry moore"` | 251 | 0 | SB2 | ? | Q3.2 / Q3.18 |  |
| B4 | Quotes after the colon: `title:"henry moore"` | 251 | 1 | SB2 | = | same as B2 |  |
| B5 | OR inside a field: `title:sculpture;landscape` | 5,120 | 0 | SA7 | ? | Q2.7 field OR as two filters |  |
| B6 | Wildcard in a field: `title:sculpt*` | 4,522 | 4,522 | SB3 | ? | Q3.10 candidate cap |  |
| B7 | Leading wildcard in a field: `title:*scape` | 921 | 0 | SB3 | ? | Q2.3 leading wildcard in a field |  |
| B8 | Another text field: `credit:wilde` | 18,879 | 18,879 | SB1 | = |  |  |
| B9 | Field name with a slash: `photographer/videographer:wilde` | 33,517 | 100,862 | SC3 | ? | Q3.12 one word of a dynamic keyword |  |
| B10 | Punctuation in a field value: `exhibitiontitle:fabric-ation` | 1,453 | 1,453 | SB4 | ? | Q2.8 / Q3.18 phrase of the parts |  |
| B11 | Dots in a field value: `dimensions:16.5x16.5cm` | 20 | 20 | SB4 | ? | Q2.8 / Q3.18 |  |
| B12 | Text field not flagged for indexing: `camera:nikon` | 0 | 0 | SK1 | ? | Q3.6 nothing in core |  |
| B13 | Multi-line field not flagged for indexing: `accessibilityalttext:island` | 0 | 0 | SK1 | ? | Q3.6 |  |
| B14 | Partially indexed field, fragment: `originalfilename:ksho` | 665 | 665 | SB6 | = | _p |  |
| B15 | Unknown field name: `foo:sculpture` | 25 | 25 | SB1 | = | unknown name → two words |  |
| B16 | NOT before a field term: `-title:sculpture` | 49,241 | 0 | SB5 | ? | Q3.8 degenerate form |  |
| B17 | NOT after the colon: `title:-sculpture` | 100,862 | 4,226 | SB5 | ? | Q3.8 excludes nothing in core |  |
| B18 | Stop word in a field: `title:the` | 100,862 | 10,564 | SB5 | ? | Q2.4 everything in core |  |
| B19 | Field hidden from advanced search, not indexed: `yearai:2025` | 0 | 0 | SK1 | ? | Q3.6 accepted by name, no keywords |  |
| B20 | Two terms, one field: `title:sculpture, title:park` | 580 | 580 | SB1 | = |  |  |
| B21 | Two fields: `title:sculpture, credit:wilde` | 316 | 316 | SB1 | = |  |  |

## C — Fixed-list fields and node tokens

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| C1 | Option by name: `orientation:landscape` | 58,028 | 58,028 | SC2 | = | resolved before the hook |  |
| C2 | Multi-word option, quoted: `"usagerights:Social media"` | 148 | 0 | SC2 | = | quoted token must be dropped from q |  |
| C3 | Multi-word option, unquoted: `usagerights:social media` | 148 | 4,763 | SC3 | ? | Q3.12 |  |
| C4 | One word of an option: `usagerights:social` | 148 | 100,862 | SC3 | ? | Q3.12 old plugin: everything |  |
| C5 | Wildcard on an option: `usagerights:soc*` | 148 | 100,862 | SC3 | ? | Q3.12 wildcard on an option |  |
| C6 | OR of options: `materials:bronze;wood` | 128 | 128 | SC2 | = |  |  |
| C7 | AND of options: `materials:bronze, materials:wood` | 0 | 128 | SC3 | ? | Q3.12 BOM-prefixed name |  |
| C8 | Dropdown option, quoted: `"source:Digital Camera"` | 2,224 | 0 | SC2 | = | non-indexed field resolves by name |  |
| C9 | One word of a dropdown option: `source:digital` | 0 | 100,862 | SC3 | ? | Q3.12 nothing in core |  |
| C10 | Checkbox option, one word: `yspactivity:learning` | 11,671 | 11,671 | SC2 | = |  |  |
| C11 | Dynamic keyword option, quoted: `"artistname:Henry Moore"` | 2,713 | 0 | SC2 | = |  |  |
| C12 | One word of a dynamic keyword: `artistname:moore` | 2,713 | 100,862 | SC3 | ? | Q3.12 |  |
| C13 | Node token: `@@389` | 58,028 | 58,028 | SC1 | = |  |  |
| C14 | OR of nodes: `@@389@@388` | 94,936 | 94,936 | SC1 | = |  |  |
| C15 | AND of nodes: `@@425, @@389` | 49 | 49 | SC1 | = |  |  |
| C16 | AND of nodes, space separated: `@@389 @@425` | 49 | 49 | SC1 | = |  |  |
| C17 | NOT a node: `@@!432` | 100,461 | 100,461 | SC1 | = |  |  |
| C18 | NOT inside an OR word: `@@!389@@388` | 94,936 | 94,936 | SC1 | = | NOT ignored inside an OR word |  |
| C19 | Unknown node: `@@999999999` | 0 | 0 | SC1 | = | unknown node → nothing |  |
| C20 | Simple-search-bar checkbox: `@@17913` | 264 | 264 | SC1 | = |  |  |
| C21 | Dynamic keyword node: `@@2997` | 33,333 | 33,333 | SC1 | = |  |  |
| C22 | GPT keyword node: `@@460` | 37,603 | 37,603 | SC1 | = |  |  |
| C23 | Option name and a node of the same field: `orientation:landscape, @@388` | 0 | 0 | SC2 | = |  |  |
| C24 | Option names as the search page converts them: `materials:bronze;wood` | 128 | 128 | SC2 | = | the page rewrites it to @@ tokens |  |
| C25 | Documented OR shape with a multi-word value: `usagerights:social media;website` | 0 | 0 | SA7 | ↩ | bare OR → suggestion in core |  |
| C26 | OR of an option name and a word: `usagerights:website;social` | 82 | 82 | SC2 | = | words discarded by core |  |

## D — Dates

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| D1 | Year: `date:2024` | 8,838 | 8,838 | SD1 | = |  |  |
| D2 | Year and month: `date:2024-05` | 572 | 572 | SD1 | = |  |  |
| D3 | Full date: `date:2024-05-31` | 317 | 317 | SD1 | = |  |  |
| D4 | Date stored with a time: `date:2017-05-10` | 626 | 626 | SD1 | = | time of day ignored |  |
| D5 | Range: `date:rangestart2024-01-01end2024-02-29` | 822 | 854 | SD3 | ? | Q3.4 |  |
| D6 | Range, start only: `date:rangestart2026-01-01` | 2,006 | 2,006 | SD3 | ? | Q3.4 |  |
| D7 | Range, end only: `date:rangeend2005-12-31` | 722 | 652 | SD3 | ? | Q3.4 |  |
| D8 | Range as the EDTF box sends it: `date:rangestart2024-00-00end2024-12-99` | 8,838 | 21,151 | SD3 | ? | Q3.4 EDTF -00 / -99 |  |
| D9 | Range, end month without a day: `date:rangestart2024-01-01end2024-02-31` | 822 | 21,151 | SD3 | ? | Q3.4 end padded -31 |  |
| D10 | Simple search year: `basicyear:2024` | 8,838 | 0 | SD2 | ? | Q3.3 / Q3.13 |  |
| D11 | Simple search month, any year: `basicmonth:05` | 9,511 | 0 | SD2 | ? | Q3.3 / Q3.13 |  |
| D12 | Simple search day, any month: `basicday:05` | 3,227 | 0 | SD2 | ? | Q3.3 / Q3.13 |  |
| D13 | Year and month dropdowns: `basicyear:2025, basicmonth:06` | 1,542 | 0 | SD2 | ? | Q3.3 / Q3.13 |  |
| D14 | Year, month and day dropdowns: `basicyear:2025, basicmonth:06, basicday:05` | 256 | 0 | SD2 | ? | Q3.3 / Q3.13 |  |
| D15 | Legacy "any year" syntax: `date:nnnn\|05` | 9,511 | 0 | SD2 | ? | Q3.3 legacy form |  |
| D16 | startdate / enddate form inputs: `startdate:2024-01-01, enddate:2024-12-31` | 0 | 0 | SD2 | · | dead inputs; core answers before the hook |  |
| D17 | Date-and-time field, full date: `expirydate:2029-05-31` | 224 | 224 | SD1 | = |  |  |
| D18 | Date-and-time field, year: `expirydate:2029` | 3,171 | 3,171 | SD1 | = |  |  |
| D19 | Date-and-time field, range: `expirydate:rangestart2029-01-01end2029-12-31` | 3,171 | 3,171 | SD3 | ? | Q3.4 |  |
| D20 | Partial value (year only): `dateofartwork:2025` | 2,593 | 2,593 | SD1 | = | year-only value |  |
| D21 | Month, including month-only values: `eventdate:2022-04` | 509 | 509 | SD1 | = | month-only value |  |
| D22 | Date field not flagged for indexing: `datephototaken:2024` | 3,676 | 0 | SD5 | ? | Q3.6 needs _q regardless of flag |  |
| D23 | Same field, full date: `datephototaken:2012-01-26` | 4,549 | 0 | SD5 | ? | Q3.6 |  |
| D24 | Date-range field, one value: `daterangetest:2020` | 0 | 2 | SD4 | ✗ | Core 2; Q3.5 |  |
| D25 | Date-range field, range: `daterangetest:rangestart2020-01-01end2021-12-31` | 1 | 2 | SD4 | ? | Q3.5 containment, not overlap |  |
| D26 | Two values for one date field: `date:2024, date:2025` | 0 | 0 | SD1 | ? | Q3.14 two values ANDed |  |
| D27 | OR on a date: `date:2024;2025` | 0 | 0 | SD1 | ? | Q3.14 OR on a date → nothing |  |
| D28 | Another date-and-time field: `copyrightexpirationd:2030` | 47 | 47 | SD1 | = |  |  |

## E — Numeric field

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| E1 | Range: `numberfield:numrange0\|100` | 9 | 9 | SE1 | = |  |  |
| E2 | Exact (one bound): `numberfield:numrange42\|` | 2 | 2 | SE1 | = | single bound = exact |  |
| E3 | Max only: `numberfield:numrange\|100` | 1 | 1 | SE1 | = |  |  |
| E4 | Negative bounds: `numberfield:numrangeneg50\|neg1` | 2 | 2 | SE1 | = |  |  |
| E5 | Decimal bounds: `numberfield:numrange3\|3.14` | 1 | 1 | SE1 | = |  |  |
| E6 | Plain number in the field: `numberfield:42` | 2 | 0 | SE1 | = | plain value → _q |  |
| E7 | numrange on a text field: `title:numrange1\|10` | 2,937 | 0 | SE1 | ? | Q3.7 numrange on a text field |  |

## F — Special searches

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| F1 | !last: `!last100` | 100 | 100 | SF1 | = |  |  |
| F2 | !last, first rows: `!last5` fetchrows=0,5 | 5 | 5 | SF1 | = | P29 |  |
| F3 | !last with no number: `!lastabc` | 1,000 | 0 | SF1 | ? | Q4.3 non-numeric → 1000 |  |
| F4 | !last plus a keyword: `!last100 sculpture` | 1,000 | 1,000 | SF1 | ? | Q4.3 space → 1000 (P28) |  |
| F5 | !last plus a node: `!last1000 @@389` | 1 | 1,000 | SF1 | ✗ | Core 1; Q11.3 |  |
| F6 | !nodownloads: `!nodownloads` | 94,493 | 0 | SF17 | ↩ | !nodownloads |  |
| F7 | !duplicates (all): `!duplicates` | 59 | 0 | SF17 | ↩ | !duplicates |  |
| F8 | !duplicates of one resource: `!duplicates123007` | 2 | 0 | SF17 | ↩ | !duplicates |  |
| F9 | !collection: `!collection3808` | 55 | 55 | SF2 | = |  |  |
| F10 | !collection, public: `!collection3618` | 949 | 949 | SF2 | = |  |  |
| F11 | !collection, featured: `!collection3706` | 262 | 262 | SF2 | = |  |  |
| F12 | !collection, featured but not public: `!collection3125` | 257 | 257 | SF2 | = |  |  |
| F13 | !collection, someone else's private: `!collection158` | 0 | 0 | SF2 | = |  |  |
| F14 | !collection, own empty collection: `!collection{owncollection}` | 0 | 0 | SF2 | = |  |  |
| F15 | !collection, own upload collection: `!collection-{userref}` | 0 | 0 | SF2 | ↩ | upload collection vetoes (P27) — core answers |  |
| F16 | !collection plus a keyword: `!collection3808 sculpture` | 55 | 55 | SF2 | = |  |  |
| F17 | !collection, comma then keyword: `!collection3808, sculpture` | 55 | 55 | SF2 | = |  |  |
| F18 | !collection plus a node: `!collection3808 @@389` | 1 | 1 | SF2 | = |  |  |
| F19 | !collection in collection order: `!collection3808` order_by=collection, sort=asc, fetchrows=0,5 | 55 | 55 | SF2 | = |  |  |
| F20 | !collection with a resource type: `!collection3808` restypes=5 | 55 | 55 | SF2 | = | restypes ignored |  |
| F21 | !collection with a workflow state: `!collection3808` archive=2 | 55 | 55 | SF2 | = | states ignored, z applies |  |
| F22 | !related: `!related121409` | 488 | 0 | SF11 | ? | Q4.1 needs related[] |  |
| F23 | !related plus a keyword: `!related121409 sculpture` | error: HTTP 0 connecti… | 0 | SF11 | ? | Q4.1 core times out |  |
| F24 | !relatedpushed: `!relatedpushed121409` | 0 | 0 | SF11 | ? | Q4.1 |  |
| F25 | !geo: `!geo53p5bm1p7t53p7bm1p4` | 4,893 | 0 | SF14 | ? | Q4.1 needs a geopoint |  |
| F26 | !colourkey: `!colourkeyEPNB` | 1,979 | 0 | SF15 | ? | Q4.1 needs colour_key |  |
| F27 | !colour: `!colourE` | 70,574 | 0 | SF15 | ? | Q4.1 |  |
| F28 | !rgb: `!rgb:610,391,405` | 500 | 0 | SF17 | ↩ | !rgb |  |
| F29 | !nopreview: `!nopreview` | 105 | 0 | SF12 | ? | Q4.1 needs has_image |  |
| F30 | !resource: `!resource416` | 1 | 1 | SF4 | = |  |  |
| F31 | !resource plus a term with digits: `!resource416, date:2024` | 0 | 1 | SF4 | ✗ | Core 3; Q4.3 |  |
| F32 | !archivepending: `!archivepending` | 0 | 0 | SF7 | ? | Q4.5 nothing for z1 users |  |
| F33 | !userpending: `!userpending` | 0 | 0 | SF7 | ? | Q4.5 |  |
| F34 | !contributions: `!contributions9` | 41,283 | 41,283 | SF5 | = |  |  |
| F35 | !contributions, own: `!contributions{userref}` | 0 | 0 | SF5 | = | P26 replicated |  |
| F36 | !images: `!images` | 100,757 | 0 | SF12 | ? | Q4.1 |  |
| F37 | !unused: `!unused` | 6,967 | 0 | SF16 | ? | Q4.7 |  |
| F38 | !list: `!list416:4460:8692` | 1 | 1 | SF3 | = |  |  |
| F39 | !listall: `!listall416:4460:8692` | 1 | 1 | SF3 | = |  |  |
| F40 | !list plus a keyword: `!list416:4460:8692 sculpture` | 1 | 1 | SF3 | = |  |  |
| F41 | !list with commas: `!list416,4460` | 1 | 1 | SF3 | = | comma ends the list |  |
| F42 | !hasdata: `!hasdata170` | 12 | 12 | SF6 | = |  |  |
| F43 | !hasdata on a non-indexed field: `!hasdata93` | 63,316 | 63,316 | SF6 | = |  |  |
| F44 | !hasdata plus a keyword: `!hasdata170 sculpture` | 0 | 0 | SF6 | = |  |  |
| F45 | !empty by ref: `!empty170` | 100,850 | 0 | SF10 | ? | Q4.8 |  |
| F46 | !empty by short name: `!emptynumberfield` | 100,004 | 0 | SF10 | ✗ | Core 5; Q4.8 |  |
| F47 | !empty on a field of one type: `!empty95` | 100,457 | 0 | SF10 | ✗ | Core 6; Q4.8 |  |
| F48 | !empty with a resource type: `!empty170` restypes=1 | 93,054 | 0 | SF10 | ? | Q4.8 |  |
| F49 | !empty, unknown field: `!emptynosuchfield` | 100,004 | 0 | SF10 | ✗ | Core 5; Q4.8 |  |
| F50 | !properties file extension: `!propertiesfext:jpg` | 66,810 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F51 | !properties, not an extension: `!propertiesfext:-jpg` | 34,052 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F52 | !properties, extension wildcard: `!propertiesfext:jp*` | 71,886 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F53 | !properties minimum height: `!propertieshmin:3000` | 69,894 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F54 | !properties, several: `!propertieswmin:4000;hmin:3000` | 55,705 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F55 | !properties minimum size: `!propertiesfmin:5` | 53,138 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F56 | !properties maximum size: `!propertiesfmax:1` | 11,993 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F57 | !properties has preview: `!propertiespi:1` | 100,072 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F58 | !properties contributor: `!propertiescu:9` | 41,283 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F59 | !properties orientation: `!propertiesorientation:portrait` | 36,092 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F60 | !properties plus a keyword: `!propertiesfext:jpg sculpture` | 32,459 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F61 | !properties plus a node: `!propertiesfext:jpg @@389` | 41,004 | 0 | SF9 | ? | Q4.1 / Q4.11 |  |
| F62 | !properties, unknown property: `!propertiesbogus:1` | 100,862 | 0 | SF9 | ? | Q4.11 unknown key → everything in core |  |
| F63 | !integrityfail: `!integrityfail` | 0 | 0 | SF13 | ? | Q4.1 |  |
| F64 | !locked: `!locked` | 1 | 0 | SF13 | ? | Q4.1 |  |
| F65 | !noningested (admin only): `!noningested` | 100,862 | 0 | SF13 | ? | Q4.1 / Q4.9 everything for non-admins |  |
| F66 | !report: `!report3p365` | 0 | 0 | SF17 | ↩ | !report |  |
| F67 | Unknown special search: `!nosuchsearch` | 100,862 | 0 | SF8 | ↩ | Q4.9 unknown special → everything in core |  |
| F68 | Two special searches: `!last100 !collection3808` | 1,000 | 0 | SF8 | = | first special only; !last1000 |  |
| F69 | !last, comma before the keyword: `!last100, sculpture` | 100 | 100 | SF1 | = | comma keeps 100 |  |

## G — Parameters

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| G1 | One resource type: `sculpture` restypes=1 | 45,727 | 45,727 | [D5](../D-search-pipeline.md) | = |  |  |
| G2 | Several resource types: `sculpture` restypes=1,5 | 48,620 | 48,620 | [D5](../D-search-pipeline.md) | = |  |  |
| G3 | Resource types "Global": `sculpture` restypes=Global | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = | Global ignored |  |
| G4 | "Global" followed by a type: `sculpture` restypes=Global,5 | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = |  |  |
| G5 | Workflow state 0: `sculpture` archive=0 | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = |  |  |
| G6 | Two states: `sculpture` archive=0,8 | 52,848 | 52,848 | [D5](../D-search-pipeline.md) | = |  |  |
| G7 | Custom state only: `sculpture` archive=8 | 3,380 | 3,380 | [D5](../D-search-pipeline.md) | = |  |  |
| G8 | State hidden by a z permission: `sculpture` archive=2 | 0 | 0 | [D5](../D-search-pipeline.md) | = | z exclusion |  |
| G9 | Pending submission: `sculpture` archive=-1 | 0 | 0 | [D5](../D-search-pipeline.md) | = | pending rule |  |
| G10 | Empty archive: `sculpture` archive= | 49,468 | 49,468 | [D5](../D-search-pipeline.md) | = | default states |  |
| G11 | Order: relevance (default): `sculpture` order_by=relevance, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | ? | Q7.1 relevance |  |
| G12 | Order: popularity: `sculpture` order_by=popularity, fetchrows=0,5 | 49,468 | 0 | SG2 | ? | Q7.2 |  |
| G13 | Order: rating: `sculpture` order_by=rating, fetchrows=0,5 | 49,468 | 0 | SG2 | ? | Q7.2 |  |
| G14 | Order: date: `sculpture` order_by=date, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | date |  |
| G15 | Order: colour: `sculpture` order_by=colour, fetchrows=0,5 | 49,468 | 0 | SG3 | ? | Q7.2 six keys |  |
| G16 | Order: title: `sculpture` order_by=title, fetchrows=0,5 | 49,468 | 0 | SG1 | ? | Q7.2 |  |
| G17 | Order: file_path: `sculpture` order_by=file_path, fetchrows=0,5 | 49,468 | 0 | SG1 | ? | Q7.2 |  |
| G18 | Order: resourceid: `sculpture` order_by=resourceid, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = |  |  |
| G19 | Order: resourcetype: `sculpture` order_by=resourcetype, fetchrows=0,5 | 49,468 | 0 | SG1 | ? | Q7.2 |  |
| G20 | Order: extension: `sculpture` order_by=extension, fetchrows=0,5 | 49,468 | 0 | SG1 | ? | Q7.2 |  |
| G21 | Order: status: `sculpture` order_by=status, fetchrows=0,5 | 49,468 | 0 | SG1 | ? | Q7.2 |  |
| G22 | Order: modified: `sculpture` order_by=modified, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | modified, ref |  |
| G23 | Order: random: `sculpture` order_by=random, fetchrows=0,5 | 49,468 | 0 | SG5 | ↩ | random never |  |
| G24 | Order: country: `sculpture` order_by=country, fetchrows=0,5 | 49,468 | 0 | SG1 | = | falls back to relevance when field 3 is absent (= G11) |  |
| G25 | Order: a field by number (date): `sculpture` order_by=field12, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | field12 is the date field |  |
| G26 | Order: numeric field: `numberfield:numrangeneg100\|2000` order_by=field170, sort=asc, fetchrows=0,12 | 0 | 0 | SG4 | ✗ | Core 8; Q7.9 |  |
| G27 | Order: title field by number: `sculpture` order_by=field8, fetchrows=0,5 | 49,468 | 0 | SG4 | ? | Q7.9 title by number |  |
| G28 | Order: unknown: `sculpture` order_by=bogus, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | unknown → relevance (= G11) |  |
| G29 | Sort ascending: `sculpture` order_by=resourceid, sort=asc, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = |  |  |
| G30 | Sort: invalid value: `sculpture` order_by=resourceid, sort=sideways, fetchrows=0,5 | 49,468 | 49,468 | [D9](../D-search-pipeline.md) | = | invalid sort → asc |  |
| G31 | Window: offset and count: `sculpture` order_by=resourceid, sort=asc, fetchrows=0,5 | 49,468 | 49,468 | [D10](../D-search-pipeline.md) | = |  |  |
| G32 | Window: count only (legacy): `sculpture` order_by=resourceid, sort=asc, fetchrows=5 | 5 rows | 5 rows | [D10](../D-search-pipeline.md) | = |  |  |
| G33 | Window: legacy with an offset: `sculpture` order_by=resourceid, sort=asc, fetchrows=5, offset=10 | 5 rows | 5 rows | [D10](../D-search-pipeline.md) | = |  |  |
| G34 | Recent days: 365: `sculpture` daylimit=365 | 10,365 | 10,365 | [D5](../D-search-pipeline.md) | = | day limit |  |
| G35 | Recent days: 90: (empty) daylimit=90 | 3,089 | 3,089 | [D5](../D-search-pipeline.md) | = |  |  |
| G36 | Recent days: 30: (empty) daylimit=30 | 0 | 0 | [D5](../D-search-pipeline.md) | = |  |  |
| G37 | go (previous / next): `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | · | go: logging only; local harness |  |
| G38 | editable_only: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | ↩ | editable_only (Q6.4); local harness |  |
| G39 | access: `(any)` | n/a | n/a | [D6](../D-search-pipeline.md) | = | P25; local harness |  |
| G40 | smartsearch: `(any)` | n/a | n/a | [D5](../D-search-pipeline.md) | ? | Q5.11 smartsearch states; local harness |  |
| G41 | ignore_filters: `(any)` | n/a | n/a | [D2](../D-search-pipeline.md) | ? | Q3.15 ignore_filters; local harness |  |
| G42 | return_refs_only / return_disk_usage / returnsql / access_override: `(any)` | n/a | n/a | [D1](../D-search-pipeline.md) | · | shapes: refs-only =, disk usage ↩, returnsql ↩, access_override = (D6/D7); local harness |  |
| G43 | Order: a field that is not a join column: `sculpture` order_by=field170, fetchrows=0,5 | 0 | 0 | SG4 | ✗ | Core 8; Q7.9 |  |

## H — Combinations

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| H1 | Keyword and node: `sculpture, @@389` | 26,893 | 26,893 | SC1 | = |  |  |
| H2 | Keyword and option name: `sculpture, orientation:landscape` | 26,893 | 26,893 | SC2 | = |  |  |
| H3 | Field term and node: `title:sculpture, @@389` | 2,439 | 2,439 | SB1 | = |  |  |
| H4 | Keyword and two nodes: `sculpture, @@425, @@389` | 41 | 41 | SC1 | = |  |  |
| H5 | Field wildcard and node: `title:sculpt*, @@389` | 2,507 | 2,507 | SB3 | ? | Q3.10 |  |
| H6 | Node and NOT node: `@@425, @@!389` | 99 | 99 | SC1 | = |  |  |
| H7 | Keyword and date: `sculpture, date:2024` | 4,517 | 4,517 | SD1 | = |  |  |
| H8 | Keyword, node and date, by date: `sculpture, @@389, date:2024` order_by=date, fetchrows=0,5 | 2,034 | 2,034 | SD1 | = | order by D9 |  |
| H9 | Keyword, type, by ref: `sculpture` restypes=5, order_by=resourceid, sort=asc, fetchrows=0,5 | 2,893 | 2,893 | [D5](../D-search-pipeline.md) | = |  |  |
| H10 | Keyword, node, type and two states: `sculpture, @@389` restypes=5, archive=0,2 | 1,408 | 1,408 | [D5](../D-search-pipeline.md) | = |  |  |
| H11 | Collection and keyword, collection order: `!collection3808 sculpture` order_by=collection, sort=asc, fetchrows=0,5 | 55 | 55 | SF2 | = |  |  |
| H12 | Last 100 and keyword: `!last100 sculpture` fetchrows=0,5 | 1,000 | 1,000 | SF1 | = |  |  |
| H13 | Node and NOT keyword: `@@425, -sculpture` | 48 | 48 | SA4 | = |  |  |
| H14 | Phrase and node: `"sculpture park", @@389` | 3,412 | 0 | SA3 | ? | Q2.5 |  |
| H15 | Keyword and NOT keyword: `sculpture, -landscape` | 15,049 | 15,059 | SA4 | = |  |  |
| H16 | Keyword and numeric range: `sculpture, numberfield:numrange0\|100` | 0 | 0 | SE1 | = |  |  |
| H17 | Node and date: `@@389, date:2024` | 4,169 | 4,169 | SD1 | = |  |  |
| H18 | OR of options and a node: `materials:bronze;wood, @@389` | 18 | 18 | SC2 | = |  |  |
| H19 | Simple-search year and keyword: `basicyear:2024, sculpture` | 4,517 | 0 | SD2 | ? | Q3.3 |  |
| H20 | Simple-search year and node: `basicyear:2024, @@389` | 4,169 | 0 | SD2 | ? | Q3.3 |  |
| H21 | Two date fields: `date:2024, expirydate:2029` | 2,972 | 2,972 | SD1 | = |  |  |
| H22 | Two wildcards: `sculpt*, land*` | 36,026 | 0 | SA5 | ? | Q2.3 two wildcards (P30 vetoed) |  |
| H23 | Phrase and NOT phrase: `"sculpture park", -"black and white"` | 6,758 | 0 | SA4 | ? | Q2.6 negated phrase (SA3 first) |  |
| H24 | Four keywords: `sculpture landscape nature outdoor` | 19,608 | 19,608 | SA1 | = |  |  |
| H25 | Two fields, node and date: `title:sculpture, credit:wilde, @@389, date:2024` | 65 | 65 | SB1 | = |  |  |
| H26 | Collection and date: `!collection3808 date:2025` | 55 | 55 | SF2 | = |  |  |
| H27 | Contributions, keyword and node: `!contributions9 sculpture, @@389` | 9,240 | 9,240 | SF5 | = |  |  |
| H28 | List and node: `!list416:4460:8692 @@389` | 1 | 1 | SF3 | = |  |  |
| H29 | Hasdata and node: `!hasdata170 @@389` | 5 | 5 | SF6 | = |  |  |
| H30 | Properties and date: `!propertiesfext:jpg date:2024` | 6,224 | 0 | SF9 | ? | Q4.1 |  |
| H31 | Everything at once: `sculpture, "sculpture park", @@389, orientation:landscape, date:2024, -"black and white"` restypes=1, archive=0, order_by=date, sort=desc, fetchrows=0,5 | 307 | 0 | SH1 | = | capstone; needs SA3 SA4 SD1 SC2 D5 D9 |  |
| H32 | Bare OR and a node: `sculpture;landscape, @@389` | 58,028 | 0 | SA7 | ↩ |  |  |
| H33 | Wildcard OR and a node: `sculpt*;land*, @@389` | 58,028 | 0 | SA7 | ↩ |  |  |
| H34 | Field OR and a node: `title:sculpture;park, @@389` | 2,653 | 0 | SA7 | ? | Q2.7 |  |
| H35 | Date range, node and keyword: `date:rangestart2024-01-01end2024-12-31, @@389, sculpture` | 2,030 | 2,034 | SD3 | ? | Q3.4 |  |
| H36 | Year as a word and a keyword: `2024, sculpture` | 5,477 | 5,476 | SA9 | ? | Q2.11 |  |
| H37 | Type name and keyword: `document sculpture` | 111 | 101 | SA9 | ? | Q2.9 |  |
| H38 | Resource types and collection ignored together: `!collection3618 sculpture` restypes=5, archive=8 | 409 | 409 | SF2 | = |  |  |

## I — Syntax edge cases

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| I1 | No space after a special search: `!collection3808,sculpture` | 55 | 55 | [D2](../D-search-pipeline.md) | = | keywords only after the first space |  |
| I2 | Comma without a space: `sculpture,landscape` | 22 | 34,409 | SA6 | ? | Q2.8 phrase in core |  |
| I3 | Comma inside a field term: `title:sculpture,landscape` | 0 | 25 | SB4 | ? | Q2.8 phrase in the field |  |
| I4 | Repeated word: `sculpture sculpture` | 49,468 | 49,468 | [D2](../D-search-pipeline.md) | = |  |  |
| I5 | Surrounding spaces: `   sculpture   ` | 49,468 | 49,468 | [D2](../D-search-pipeline.md) | = |  |  |
| I6 | Trailing colon: `sculpture:` | 49,468 | 49,468 | SB1 | = | unknown name → the colon becomes a space |  |
| I7 | Leading colon: `:sculpture` | 49,468 | 49,468 | SB1 | = | empty name → same as I6 |  |
| I8 | Unbalanced quote: `"sculpture` | 49,468 | 0 | SA3 | = | unbalanced quote is a separator |  |
| I9 | Upper-case special search: `!LAST100` | 100,862 | 1,000 | SF8 | ↩ | Q4.9 case-sensitive → unknown |  |
| I10 | Nodes without a space: `@@389,@@425` | 49 | 49 | SC1 | = |  |  |
| I11 | Upper-case field name and value: `Title:Sculpture` | 4,226 | 4,226 | SB1 | = |  |  |
| I12 | Dropdown value from the advanced form: `"source:igital Camer"` | 0 | 0 | SJ1 | · | unreachable branch |  |

## J — Strings as the advanced search page assembles them

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| J1 | Resource IDs, words, a field, nodes, type and state: `!list2181:3879:4093 sculpture, landscape, title:sculpture, @@389@@388` restypes=5, archive=0 | 1 | 1 | SJ1 | = |  |  |
| J2 | The same with a property filled in: `!propertiesfext:jpg ,!list2181:3879:4093 sculpture, landscape, title:sculpture, @@389@@388` restypes=5, archive=0 | 8 | 0 | SJ1 | ? | Q4.10 |  |
| J3 | Posted special search prefix: `!contributions9,sculpture` | 41,283 | 41,283 | SJ1 | = | I1 rule |  |
| J4 | Posted special search prefix, more terms: `!contributions9,title:park, sculpture` | 19,396 | 19,396 | SJ1 | = |  |  |
| J5 | Range with a start year only: `date:rangestart2024` | 21,151 | 21,151 | SD3 | ? | Q3.4 unpadded year |  |
| J6 | Range with months only: `date:rangestart2024-05end2024-06-31` | 1,947 | 19,079 | SD3 | ? | Q3.4 |  |
| J7 | Live result count: `sculpture` fetchrows=0,0 | 49,468 | 49,468 | [D10](../D-search-pipeline.md) | = | count only |  |
| J8 | Checkbox list, two options ticked: `sculpture, @@6125@@6130` | 284 | 284 | SC2 | = |  |  |
| J9 | Checkbox list with $checkbox_and: `sculpture, @@6125, @@6130` | 0 | 0 | SC2 | = |  |  |
| J10 | Dropdown choice: `sculpture, @@247` | 1,429 | 1,429 | SC2 | = |  |  |
| J11 | Property filter with terms: `!propertiesfext:jpg ,sculpture, title:park` | 547 | 0 | SJ1 | ? | Q4.1 properties then terms |  |
| J12 | Full-text box with a word: `sculpture, "@FULL_TEXT:distant island"` | 461 | 0 | SA8 | ↩ |  |  |
| J13 | Date value and range on one field: `date:2024-05, date:rangestart2024-05-01end2024-05-31` | 524 | 572 | SD3 | = | a value (SD1) and a range on one field, ANDed |  |

## K — What the indexing flags control

| Row | Search | Core | Old | Owner | Plan | Note | Done |
|---|---|---|---|---|---|---|---|
| K1 | Word that exists only in a non-indexed field: `nikon` | 0 | 0 | SK1 | · | core answers before the hook |  |
| K2 | The same word with a wildcard: `nikon*` | 23,412 | 0 | SK1 | ? | Q1.1 / Q2.3 |  |
| K3 | Wildcard inside the non-indexed field: `camera:nik*` | 23,412 | 0 | SK1 | ? | Q1.1 |  |
| K4 | Full-text box on a non-indexed field: `"@FULL_TEXT:nikon"` | 23,412 | 0 | SA8 | ↩ |  |  |
| K5 | Bare fragment of a partially indexed field: `ksho` | 665 | 665 | SB6 | ? | Q1.3 fragment as a bare word |  |
| K6 | Word in an inactive field: `description:sculpture` | 0 | 57 | SK2 | = | P37 |  |
| K7 | Phrase held only in non-indexed fields: `"library book"` | 0 | 0 | SK1 | · | core answers before the hook |  |
| K8 | The same phrase through full text: `"@FULL_TEXT:[QUOTES]library book[QUOTES]"` | 1 | 0 | SA8 | ↩ |  |  |
| K9 | Complete indexing: `field:two words` | n/a | n/a | SK3 | ? | Q1.12 no such field on the test system |  |
| K10 | Word beyond the 500th character: `field:word` | n/a | n/a | SK3 | ? | Q1.13 |  |
| K11 | Number range on a non-indexed numeric field: `field:numrange1\|10` | n/a | n/a | SE1 | ? | Q3.6 no such field on the test system |  |
| K12 | Field hidden from the user by permission: `field:word` | n/a | n/a | SK2 | = | core returns false before the hook; local harness |  |

## Totals

| Plan | Rows |
|---|---|
| = | 145 |
| ? | 120 |
| ↩ | 22 |
| ✗ | 8 |
| · | 8 |

Rows per owner: D1 3 · D2 4 · D5 16 · D6 1 · D9 8 · D10 4 · SA1 7 · SA2 2 · SA3 5 · SA4 5 · SA5 5 · SA6 3 · SA7 7 · SA8 7 · SA9 6 · SA10 2 · SA11 2 · SA12 1 · SB1 10 · SB2 3 · SB3 3 · SB4 3 · SB5 3 · SB6 2 · SC1 14 · SC2 14 · SC3 7 · SD1 15 · SD2 9 · SD3 10 · SD4 2 · SD5 2 · SE1 9 · SF1 7 · SF2 16 · SF3 5 · SF4 2 · SF5 3 · SF6 4 · SF7 2 · SF8 3 · SF9 14 · SF10 5 · SF11 3 · SF12 2 · SF13 3 · SF14 1 · SF15 2 · SF16 1 · SF17 5 · SG1 6 · SG2 2 · SG3 1 · SG4 3 · SG5 1 · SH1 1 · SJ1 6 · SK1 7 · SK2 2 · SK3 2
