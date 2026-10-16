# Search work packages, by catalogue group

The search side of the plugin is cut to the **core search catalogue**: every form of search core's
`do_search()` accepts, one row each (`docs/core-search-catalogue.md` on the
`typesense_search_catalogue` branch of `cwe/resourcespace`, groups A–K, 300 rows measured live). A
copy with a **Required** column for deciding which cases the rewrite must serve is
[../../catalogue.md](../../catalogue.md); its marks feed the planned disposition of every row in
[coverage.md](coverage.md).
Each package below owns a set of rows, has its own fixture additions and its own harness battery,
and can be built and proved on its own once the pipeline (D1–D11) exists. The row-by-row map, with
what the plan intends for each row, is [coverage.md](coverage.md).

Package IDs are `S` + the catalogue group letter + a number, so `SB3` is the third package of
catalogue group B. (The code-area letters A–I in [../README.md](../README.md) are a different
alphabet; the `S` prefix keeps them apart.)

| Group | File | Packages | Milestone |
|---|---|---|---|
| A Keywords | [A-keywords.md](A-keywords.md) | SA1 plain words · SA2 stop words · SA3 phrases · SA4 negation · SA5 wildcards · SA6 punctuation · SA7 OR groups · SA8 full-text box · SA9 non-metadata words · SA10 related keywords · SA11 unknown words · SA12 stemming | SA1, SA11, SA12 M2; rest M6 |
| B Field text | [B-field-text.md](B-field-text.md) | SB1 lookup and `field:word` · SB2 quoted and multi-word · SB3 wildcards in a field · SB4 punctuation in a value · SB5 negation and stop words · SB6 partial indexing | M4 |
| C Fixed list and nodes | [C-fixed-list-and-nodes.md](C-fixed-list-and-nodes.md) | SC1 node tokens · SC2 options core resolves · SC3 option text core does not resolve | SC1, SC2 M2; SC3 M4 |
| D Dates | [D-dates.md](D-dates.md) | SD1 values · SD2 dropdowns and legacy forms · SD3 ranges · SD4 date-range fields · SD5 non-indexed date fields | M4 |
| E Numeric | [E-numeric.md](E-numeric.md) | SE1 `numrange` and plain numbers | M4 |
| F Specials | [F-specials.md](F-specials.md) | SF1 `!last` · SF2 `!collection` · SF3 `!list` · SF4 `!resource` · SF5 `!contributions` · SF6 `!hasdata` · SF7 pending · SF8 unknown specials · SF9 `!properties` · SF10 `!empty` · SF11 `!related` · SF12 `!images` · SF13 flags · SF14 `!geo` · SF15 colour · SF16 `!unused` · SF17 stays on core | SF8 M2; SF1–SF7, SF17 M5; SF9–SF16 M7 |
| G Parameters | [G-parameters.md](G-parameters.md) | SG1 text-key sorts · SG2 rating and popularity · SG3 colour sort · SG4 `field<n>` sorts · SG5 random (the rest of group G is D1, D5, D9, D10, D11) | SG5 M2; rest M7 |
| H Combinations | [H-combinations.md](H-combinations.md) | SH1 the combinations gate | M2 onward |
| I, J Syntax and forms | [IJ-syntax-and-forms.md](IJ-syntax-and-forms.md) | group I is D2's parse battery · SJ1 strings as the pages assemble them | M4 |
| K Indexing flags | [K-indexing-flags.md](K-indexing-flags.md) | SK1 non-indexed fields · SK2 inactive and hidden fields · SK3 complete indexing and the 500-character cap | M4 |

54 packages. With D1–D12 the search side is 66 tasks.

## Package format

The task format of [../README.md](../README.md) plus four lines, so a package can be read on its
own and tested on its own:

- **Rows** — the catalogue rows it owns, and the rows of other packages it is also a case for
  (*cross*). A row has exactly one owner ([coverage.md](coverage.md)).
- **Core** — what core does for those rows, in one paragraph, read from the catalogue and the
  code. The package reproduces this, declines it, or records a difference — never guesses.
- **Serve** — the Typesense translation, with the verified facts it rests on.
- **Fixture** — what the local battery needs beyond the base fixture. Additions are registered in
  the fixture library (`fixture_lib.php` helpers) and used only by the batteries that need them,
  so a package's cases do not depend on another package's data (Q10.13).
- **Battery** — one harness file per package, `harness/ab/<ID>_<slug>.php`, with case labels that
  carry the catalogue row IDs (`A7`, `A7.2` for a variant, `x1` for an extra case) so
  `summarise.php` can fill the catalogue's *OK* column from a run; and the live rows to send with
  `catalogue_run.php <IDs>` (Q10.12).

The remaining lines are the usual ones: *Builds*, *Size*, *Depends on*, *Milestone*, *Must honour*,
*Blocking questions* (from the bank — answered before the package starts), *Open questions* (added
by this plan to fill the package out — answered as part of starting it).

## How a package is proved

1. Its battery runs green locally under the acceptance rule (Q10.3): every owned row matches core
   in set, total and order, or declines with its planned reason, or differs only in relevance-to-text
   order and is listed with a decision row that names its bin (documented behaviour, incidental
   behaviour, core bug — principle 11). Expected sets are derived by hand from the fixture, never
   copied from a live total.
2. [SH1](H-combinations.md) still agrees with its term pool enlarged by the package's terms.
3. The owned rows are sent live (`catalogue_run.php`) and the catalogue table regenerated; the
   *Done* column in [coverage.md](coverage.md) is ticked.
4. Any fact learned goes into the package's *must honour* and, if users or administrators need
   it, the plugin docs.

## Fixture additions the packages need

The base fixture (`ab/fixture_body.php`: 29 resources, 15 fields, three users, four types) already
has a dropdown, a dynamic keyword list, a category tree, a partial-indexed field, a numeric field,
a date-range field, an HTML field, a hidden field, an inactive field, a non-indexed multi-line
field, a translated option, a year-only date, a 600-character value, and access/grant cases. The
packages add:

| Addition | Needed by |
|---|---|
| A multi-value field whose phrase spans two values | SA3 |
| The 40-completion word set and a 60-prefix set (from batteries 23 and 27) | SA5, SB3, SA1 |
| A resource whose ref is a word elsewhere; type names and usernames as values | SA9 |
| A related-keyword pair in both directions | SA10 |
| A `credit` field; a field whose short name contains a slash | SB1 |
| An option with a leading byte-order mark; an indexed and a non-indexed fixed-list field | SC3, SK1 |
| 600- and 1,000-id node buckets; a 41-node chain (generated) | SC1 |
| A month-only date value; two values on one resource; boundary values for every range form | SD1, SD3 |
| A non-indexed date field and a non-indexed numeric field | SD5, SE1, SK1 |
| A date-range pair spanning a year boundary | SD4 |
| Negative, zero and duplicate numeric values | SE1 |
| Collections: own, public, featured, another user's private, upload, share key, upload-share session; a smart collection | SF2 |
| File extension, size, dimensions, preview flag, colour key, a geopoint, `lock_user`, `integrity_fail`, related links | SF9–SF16 |
| Ties: equal titles, empty titles, NULL extensions, equal modified dates; known hit counts and ratings | SG1, SG2, D9 |
| A complete-indexed field with one-word and two-word values | SK3 |
| The factorial fixture (512 resources, nine attributes) with the term pool per milestone | SH1 |

## What the packages do not cover

Permissions, access and read-time validation (D5–D8, C8), freshness (C), windows and shapes (D10,
D11) are pipeline tasks with their own batteries — the catalogue's two test users see every field
and every collection, so those batteries use the harness's user matrix rather than catalogue rows.
