# SH — Combinations (catalogue group H)

Terms of different kinds in one search: everything is ANDed — keyword unions, phrase unions, node
buckets, date and numeric joins, the special's own filter and the parameters. Each H row is owned by
the **last** package it needs (see [coverage.md](coverage.md)); this package is the invariant that
the rows together prove, run as a gate.

### SH1 — The combinations gate
- **Rows:** H31 (the capstone) · every other H row is listed here and owned elsewhere: H1 H4 H6
  (SC1) · H2 H18 (SC2) · H3 H25 (SB1) · H5 (SB3) · H7 H8 H17 H21 (SD1) · H9 H10 (D5) · H11 H26 H38
  (SF2) · H12 (SF1) · H13 H15 (SA4) · H14 (SA3) · H16 (SE1) · H19 H20 (SD2) · H22 (SA5) · H23 (SA3)
  · H24 (SA1) · H27 (SF5) · H28 (SF3) · H29 (SF6) · H30 (SF9) · H32 H33 H34 (SA7) · H35 (SD3) ·
  H36 H37 (SA9)
- **Builds:** `harness/ab/SH1_combinations.php` — the factorial battery (512 resources, nine
  two-valued attributes, ~2,500 generated cases, each checked three ways: core against the
  expected set, the plugin against the expected set, the two against each other), regenerated so
  its term pool grows with each merged package; and the explicit H rows as a live list.
- **Size:** M once, then S per package · **Depends on:** H1, D1–D11 · **Milestone:** M2 onward
- **Invariant:** terms that agree on their own agree in combination. Only terms whose own
  package is green go into the pool, so a failure here comes from the combination (the shared
  keyword step, the filter compiler, restriction suppression, the recent-N selection) — not from
  a term.
- **Must honour:** the one disagreeing form among 2,481 cases on the old plugin was recorded in
  the review — carry it in as a case; `!last` + anything exercises the cutoff query's filters;
  `!collection` + anything exercises restriction suppression; `!contributions<self>` + anything
  exercises the all-off path (P26); a filter-operations overflow in a combination must veto, not
  return 0 (P34); H31 orders by date with `restypes` and `archive` set — the parameters are in the
  pool too.
- **Gate rule:** a package merges only when its own battery is green **and** SH1 with the
  enlarged pool is green (Q10.15 decides whether this is per package or per milestone, given the
  ~2.5 minutes a full run takes; `HARNESS_COMBO_SCALE` shrinks it).
- **Battery:** the file above; live, the H rows after each milestone from M4.
- **Blocking questions:** Q10.3
- **Open questions:** Q10.15
