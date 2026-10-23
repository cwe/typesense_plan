# SI / SJ — Syntax edge cases and the strings the pages assemble (catalogue groups I and J)

Group I (strings a user might type that core reads differently from what was meant) is the parse
battery of D2 — see [coverage.md](coverage.md) for the row-by-row ownership. Group J is one
package: the strings `search_form_to_search_query()` and the search bar build, tested end to end
from form input rather than from hand-typed text.

**Catalogue marks (2026-10-20):** I1 is *no* (a special followed by a comma and no space is declined; core
answers); I3, I6, I7, I8 and I12 are *check*; I2 and I9 are bug rows; I4, I5, I10 and I11 are *must*. J4, J9
and J13 are *check*; the rest of J is *must*, J2 as a bug row. One tension to resolve: J3
(`!contributions9,sculpture`, as the form posts it) is *must* while I1 (`!collection3808,sculpture`) is *no*,
and both are the same parse rule — one disposition has to win (decisions.md).

### SJ1 — Strings as the pages assemble them
- **Rows:** J1 J2 J3 J4 J11 I12 (owned) · J5 J6 J13 (SD3/SD1) · J7 (D10) · J8 J9 J10 (SC2) · J12
  (SA8) · the search bar's rebuilt text (harness 25)
- **Builds:** `harness/ab/SJ1_form_strings.php` — cases that start from **form inputs** (the
  advanced search's boxes, the simple bar's dropdowns and checkbox, a clicked metadata value, a
  clicked node) and go through core's own assembly before `do_search()`; plus any parse rule the
  cases show D2 is missing.
- **Size:** S · **Depends on:** D2, SC2, SB2, SF3, SF9 · **Milestone:** M4 (grows to M7)
- **Core:** the form builds one string in a fixed order: the date dropdowns, the words of the
  all-fields box, a `!list` prefix for the resource IDs box, the full-text box, one term per word
  per field box (a quoted phrase stays quoted around the whole term: `"title:henry moore"`), the
  ticked options and dropdown choices as `@@` tokens, and last a `!properties` prefix joined with
  " ," (J11); a posted special such as `!contributions` is prefixed with a comma and no space, so
  the term after it is dropped (J3, J4, the I1 rule); a checkbox list with every option ticked is
  omitted, the Featured Collections type and the access selector are PHP arguments, not text;
  fields with a verbatim-keyword regex keep their value unsplit; the dropdown branch that strips
  the first and last characters of a value with spaces is unreachable (I12); the search bar
  rebuilds a clicked value as `"field:Two Words"` or `subject:Birds` and a clicked node as text
  (harness S2, S3, S7, S8); `pages/search.php` rewrites `field:a;b` option names into `@@a@@b`
  (C24) and passes `[offset − colcount, n]` when collections sit above results (Q7.5).
- **Serve:** nothing of its own — the package proves the parse (D2) and the mechanism packages
  cover what the pages send, which is the only way users reach most forms.
- **Must honour:** the `!properties`-before-`!list` collision (J2, Q4.10) is a core behaviour the
  plugin can only replicate or raise; the live count (`fetchrows=0,0`, J7) is sent on every
  keystroke of the advanced form.
- **Fixture:** the base fixture; the cases call `search_form_to_search_query()` with arrays
  shaped like the posted form.
- **Battery:** `harness/ab/SJ1_form_strings.php` — J1–J4, J11, I12 and the search bar's rebuilt
  text (harness 25), each case built from form input; live: the J rows.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** only text after the first space survives a special prefix (`ds:125-133`); the user ref of `!contributions9,title:park, sculpture` survives as `9,title:park,` coerced to 9 by the `i` binding (`sf:1438-1458`); with `$checkbox_and` the form sends `word, , @@a, @@b`, each token its own AND bucket; the dropdown branch at `sf:280-291` is unreachable (dropdowns post `nodes_searched`); the `startdate` / `enddate` inputs are dead.
- **Blocking questions:** Q3.2, Q4.10
- **Open questions:** Q10.14 (the forms the API cannot send are covered locally)
