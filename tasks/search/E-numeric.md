# SE — Numeric fields (catalogue group E)

A text field with `field_constraint = 1` takes `shortname:numrangeMIN|MAX` (`neg` is a minus
sign; one bound is an exact value). One package, milestone M4.

### SE1 — `numrange` and plain numbers
- **Rows:** E1 E2 E3 E4 E5 E6 E7 H16 K11
- **Builds:** `field:numrange<min>|<max>` → `field_<ref>_f:[min..max]`; a single bound →
  `field_<ref>_f:=<n>`; `field:<number>` → `field_<ref>_q:=<number>`; `neg` → `-`.
- **Size:** S · **Depends on:** SB1, B2 · **Milestone:** M4
- **Core:** `name >= 0 AND name <= 100` as a numeric comparison (E1: 9); one bound is exact —
  `name = max(min, max)` (E2 `42|` 2, E3 `|100` 1); `neg50|neg1` is −50 to −1 (E4); decimals
  compare as numbers (E5); `numberfield:42` is not a range but an ordinary keyword search for "42"
  in the field (E6: 2); `numrange` is accepted on **any** type-0 text field and compares its
  values as numbers (`title:numrange1|10` 2,937 — titles that are numbers between 1 and 10, with
  MySQL's string-to-number cast); a numeric field that is not flagged for indexing still works
  (node join, K11).
- **Serve:** `_f` for the range and single-bound forms (verified live on field 170, 8/8, and in
  composition with keywords, sorts, restypes and `!last`); `_q` for the plain value (filtering
  `_s` gave 0); for `numrange` on a non-numeric field: veto or replicate MySQL's cast (Q3.7);
  `_f`/`_q` indexed for every numeric field regardless of flag (SD5's rule).
- **Must honour:** `_f` was almost never written before the `field_constraint` fix (a `$value
  == 1` test); a value that is not numeric (`abc`, resource 16) goes to `_s` only and must not
  match any range; `field_constraint == 1` is also what the search UI's number widget and core's
  `+0` sort key off (SG4); the field is single-line only.
- **Fixture:** the base fixture's price field 90 (100, 50, 12.5, 250, abc); add a negative, a
  zero and a duplicate value (the live field-170 set).
- **Battery:** `harness/ab/SE1_numeric.php` — E1–E7, H16, K11, harness B40, B44, N11a, O3.
- **Blocking questions:** Q3.7 (the non-numeric-field case and whether to add open-ended
  comparisons)
