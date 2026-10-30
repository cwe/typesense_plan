# SD — Dates (catalogue group D)

A date field (date, date-and-time, expiry, date-range) takes a value or a range after the colon.
Core matches a value as a **prefix on the stored string** and compares ranges **as strings**;
`basicyear`/`basicmonth`/`basicday` address `$date_field` from the simple search bar. Five packages,
milestone M4; SD3 is the one with an open design decision (Q3.4).

**Catalogue marks (2026-10-20):** SD1 required (13 must, 2 check); SD2 required (8 must, 1 check); SD3 required (5 must, 5 check); SD4 pending check (1 check, 1 bug); SD5 pending check (2 check). The check rows are in
[checks.md](checks.md) under *Dates*; SD4 and SD5 are not started until theirs are done.

**Catalogue marks (2026-10-30):** every check row decided — SD1 14 must (D27 *no*); SD2 8 must (D16 *no*: dead
inputs); SD3 10 must (Q3.4 (a) reproduces the string ranges); SD4 required — D25 *must* with the overlap rule and
an array `_k` (schema addition), D24 a bug row; SD5 required — D22 and D23 *must* (every date field has `_q` and `_k`).

### SD1 — Date values
- **Rows:** D1 D2 D3 D4 D17 D18 D20 D21 D26 D27 D28 H7 H8 H17 H21 · cross: J13 (SD3: a value and a range on one field)
- **Builds:** `field:<YYYY | YYYY-MM | YYYY-MM-DD>` → `field_<ref>_q:=<value>` (P32) for every date
  field type.
- **Size:** S · **Depends on:** SB1, A4 · **Milestone:** M4
- **Core:** `LIKE "<value>%"` on the field's node names, so a year, a month or a day all match,
  including values stored with a time (D4, D23) and partial values stored as `2025-00-00` /
  `2022-04-00` (D20, D21); two values on one field are two joins ANDed — only resources holding
  both (D26, 0 on the test system); `date:2024;2025` is `LIKE "2024;2025%"`: nothing (D27).
- **Serve:** exact match on the `_q` representation array, which holds the value's year,
  year-month and full date (and the date part only of a value with a time); the index must
  contain `_q` for every date field whatever its index flag (SD5); D26 as two ANDed filters; D27
  as nothing (a `;` in a date value → veto or an impossible filter, Q3.14).
- **Must honour:** `date:2024` once matched the time `20:24` through the bare-colon operator —
  always `:=` on `_q` (P32); a year-only value `2024-00-00` yields the representation `2024` and
  nothing finer; `_q` is never stemmed; the field named `date` is a real field on the test system.
- **Fixture:** the base fixture (date field 12 with full, year-only and time-bearing values,
  expiry field 95); add a month-only value and a second value on one resource.
- **Battery:** `harness/ab/SD1_date_values.php` — the rows above, harness I10, O17, N3a, N3b.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** a date value is `dn.name LIKE 'value%'` (`dsk:146-158`) — a value stored with a time matches a date prefix (81,866 of 82,025 values on the second database carry a time); the index flag is never read; `date:2024;2025` reaches the date branch before the OR handling and finds nothing; a value and a range on one field are three ANDed joins.
- **Blocking questions:** — (P32 confirmed)
- **Open questions:** Q3.14 (two values and `;` on a date field)

### SD2 — The date dropdowns and the legacy forms
- **Rows:** D10 D11 D12 D13 D14 D15 D16 H19 H20
- **Builds:** `basicyear:` / `basicmonth:` / `basicday:` against `$date_field`; `date:nnnn|05`.
- **Size:** S · **Depends on:** SD1, B2 (the date-component entries) · **Milestone:** M4
- **Core:** the simple search bar's dropdowns send `basicyear:2024` (one join on `$date_field`,
  `LIKE "2024-__-__%"`, 8,838), `basicmonth:05` (`LIKE "____-05-__%"`: any year, 9,511),
  `basicday:05` (`LIKE "____-__-05%"`, 3,227), and combinations (`2025-06-__%`, `2025-06-05%`);
  `basicday` is offered only with `$searchbyday` but handled regardless; the legacy
  `date:nnnn|05` turns `n` into `_` and `|` into `-` (`LIKE "____-05%"`); `startdate:` /
  `enddate:` are assembled by the form code but nothing handles them — unknown names, so words
  and a suggestion (D16, never reaches the plugin).
- **Serve:** the natural mapping is three integer fields on the resource for `$date_field` only
  (`date_year`, `date_month`, `date_day`, absent when the component is 0): `basicyear:2024` →
  `date_year:=2024`; the three combine as ANDs; `nnnn|05` → `date_month:=5`. The alternative is
  month-only and day-only tokens in `_q` (`m05`, `d05`, `2025d05`) — more tokens, no new fields.
- **Must honour:** these are not field names — the lookup must special-case them before SB1
  treats `basicyear` as an unknown field (today 8,838 / 0); a day with `basicyear` but no month
  (`2024-__-05`) is a valid core form; `$date_field` may change between rebuilds (B9, E26).
- **Fixture:** the base fixture's field 12 values (years 2019–2024, months, days).
- **Battery:** `harness/ab/SD2_date_dropdowns.php` — D10–D16, H19, H20, harness B70–B74, N2a–N2c.
- **Blocking questions:** Q3.3
- **Open questions:** Q3.13 (integer components or `_q` tokens)

### SD3 — Date ranges
- **Rows:** D5 D6 D7 D8 D9 D19 J5 J6 J13 H35
- **Builds:** `field:rangestart<A>end<B>`, either bound optional, with the comparison rule Q3.4
  decides.
- **Size:** M · **Depends on:** SD1, A4 · **Milestone:** M4
- **Core:** `name >= start AND name <= "end 23:59:59"` as **strings**: a year-only start
  (`2024`, J5) means everything from "2024" on; the advanced form pads a missing day with `31`
  (`2024-02-31`, D9: 822) and a missing month with nothing (`2024-05`, J6); the EDTF box sends
  `2024-00-00` … `2024-12-99` (D8: 8,838 = the year); a stored year-only value `2020-00-00` sorts
  before `2020-01-01`, so it never matches a range starting inside its year.
- **Serve:** (a) replicate the string comparison with a fixed-width sortable key — the padded
  value's digits as an integer (`20240231`, `20200000`) in a `_k` field, filtered numerically, so
  every form above gives core's answer including the non-real ends; or (b) interval overlap on
  `_range_start`/`_range_end` (P5: more correct, +106 on the test system, the behaviour doc
  describes it; non-real ends must be clamped). Q3.4 decides; the recommendation is (a) when the
  goal is parity (Q0.2) because the advanced form produces these ends routinely.
- **Must honour:** the old plugin dropped a non-real end and so returned the open range (822 /
  21,151); `rangestart`/`rangeend` with no value is a valid form (D6, D7); a range on a
  date-and-time field compares the same way (D19); both bounds go through A4, which must keep
  the raw digits for the (a) key and not reject `2024-02-31`; `date:rangestart2024` has no `-`
  (J5).
- **Fixture:** values on either side of each boundary form (year-only, month-only, a 29 February,
  a value with a time at 23:30 on the end day).
- **Battery:** `harness/ab/SD3_date_ranges.php` — the rows above, harness B53–B57, N4a–N4c, the
  `_range_*` vs `_k` figures recorded for the decision.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** ranges are string comparisons `name >= start` and `name <= end 23:59:59` (`dsk:179-203`), the form pads a blank day with `-31` (`sf:3369`) and the EDTF box with `-00` / `-99` (`sf:3334-3344`); `$rangestart` / `$rangeend` are never reset between keywords (`dsk:183-190`), so a later end-only term inherits an earlier start — a core quirk for Q11.3, not replicated.
- **Blocking questions:** Q3.4

### SD4 — Date-range fields (start/end pairs)
- **Rows:** D24 D25
- **Builds:** `field:<value>` and `field:range…` on a `FIELD_TYPE_DATE_RANGE` field.
- **Size:** S · **Depends on:** SD1, SD3 · **Milestone:** M4
- **Core:** a single value is meant to find ranges containing it but the parameters are bound in
  the wrong order, so nothing (D24, Core 2); a range matches when some node ≥ start and some node
  ≤ end as strings — overlap, since the two nodes of the pair are not told apart (D25: 1).
- **Serve:** decided 2026-10-30 (Q3.5): the range form through an array `_k` holding both nodes
  (`_k:>=S && _k:<=E`, a schema addition); the single-value form stays a bug row (D24, Q11.3).
  The fixture covers partial overlap, enclosing, outside and open-ended ranges.
- **Fixture:** the base fixture's field 91 (`2024-03-01` … `2024-03-10` on resource 14); add a
  pair spanning a year boundary.
- **Battery:** `harness/ab/SD4_date_range_fields.php` — D24, D25, harness B58, 16_core_bugs "the
  stored start date" / "year of the stored range".
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** a date-range field's two nodes are indistinguishable to the joins, so the rule is **overlap** — some node ≥ start and some node ≤ end — not containment (`dsk:179-203`, `def:78-79`, `rf:856-857`); an enclosing range matches; `_k` stored as an array reproduces it (`_k:>=S && _k:<=E`); no such field exists on the second database, so the fixture carries it.
- **Blocking questions:** Q3.5, Q11.3

### SD5 — Date fields not flagged for indexing
- **Rows:** D22 D23 · cross: K11 (SE1), SK1
- **Builds:** the rule that date (and numeric) representations are indexed for every field of
  those types regardless of `keywords_index`.
- **Size:** S · **Depends on:** B1, B2 · **Milestone:** M4 (needs a rebuild)
- **Core:** date terms join the node table directly, so the index flag does not matter
  (`datephototaken:2024` 3,676; the old plugin had no attributes for the field → 0).
- **Serve:** B2's selector gives `_q` (and `_k` or `_range_*`) to every date field;
  `query_by` still excludes the field for plain words (core's keyword table has none of its
  words). This is a slice of Q1.1 that can be decided on its own.
- **Fixture:** a date field with `keywords_index = 0` (as field 96 is for text).
- **Battery:** `harness/ab/SD5_non_indexed_dates.php` — D22, D23, harness O1, O2.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** the date branch never reads `keywords_index` and nodes exist for every value (`nf:121`, `145`, `rf:2783-2790`), so Q1.1 (b) and `_q` regardless of flags cover it; every date field on the second database is indexed, so the case is fixture-only there.
- **Blocking questions:** Q3.6, Q3.16
