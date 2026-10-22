# H — Verification

Milestone M0 for H1, then one battery per task as the tasks land. The harness is the one piece of
existing code the rebuild is likely to keep (Q10.11).

### H1 — Harness foundation in the repo
- **Builds:** `typesense_plan/harness/` — the planning repository, not the plugin (Q12.6 decided (b),
  2026-10-22, provisional): `boot.php` (finds the ResourceSpace checkout through a path setting;
  SQLite from its `dbstruct/`, the real core `*_functions.php` with the SQL translations, a private
  local Typesense), `fixture_lib.php`,
  the A/B runner (`do_search()` with the plugin off then on; ref sets, totals, order, row keys), the
  **accepted-divergence list** the runner reads, `run_all.sh`, `summarise.php`; and one file,
  `plugins/typesense_search/tests/<n>_typesense_smoke.php`, that core's `tests/test.php` scans: the
  bridge to the external harness — it runs the smoke battery when the harness path is configured and
  a local Typesense is reachable, and skips otherwise (Q10.2 decided (b), 2026-10-21 — no CI job;
  `composer dev:beforeCommit` is the gate, as for core).
- **Size:** M · **Depends on:** —
- **Must honour:**
  - Not `tests/` for the harness itself — core's test runner scans `plugins/*/tests` (P39), which is
    now the point: the one-file bridge lives there, the harness does not. Location (Q10.1 as amended
    by Q12.6): the planning repository, outside the release; H1's first act is moving the harness
    files from the `typesense_search_harness` branch into it (the branch itself is Chris's to merge
    or drop); `results/` is not committed again — only the generated catalogue table.
  - Gotchas: define `RS_TEST_MODE` or `metadata_field_view_access()` returns true on the CLI; core
    `exit()`s on an invalid search-filter id; a fixture word must exist in the keyword table or core
    returns before the hook; the veto reason is read from the debug file.
  - SQL translations: full text emulated (mark its results as emulated), `RLIKE` → `REGEXP`, `UNION`
    parentheses, `curdate()`, `IF` → `iif`, numeric compares `CAST AS REAL`, `ORDER BY ref DESC
    LIMIT` → `r.ref`, `get_nodes()` answered directly, `COLLATE NOCASE` for `_ci`; accents are not
    emulated.
  - A fact a case reports as "same" must be checked for the option really being applied (a runner
    once merged options with `array_merge` and six cases were wrongly recorded as matched).
  - **The acceptance rule (Q10.3):** a served row passes when set, total and order equal core's
    (order only under an explicit sort, or by the composite key for keyword-less relevance);
    relevance-to-text order is the one listed kind of difference; a declined row passes only with
    its planned reason; core-bug rows run as expected-to-differ (H7); security cases are never
    listed; everything else unlisted fails. The list is one entry per row or case — the kind of
    difference, the reason, the decision row — and a check in the planning repository confirms each
    entry matches `coverage.md`.
  - **Reuse, reviewed (Q10.11):** `boot.php`, `fixture_lib.php`, the A/B runner, `summarise.php`, the
    live and catalogue runners come across; the three couplings to the old plugin are rewritten
    (the fixture index is built by B8's rebuild, the veto reason comes from G2's log line, the config
    names are the new ones); the batteries are re-cut per package as packages land; `trace/` is
    retired; `results/` is cleared. Core's side of a wildcard or full-text case is an emulation
    until H9.
- **Blocking questions:** Q10.1 (decided), Q10.2 (decided), Q10.3 (decided), Q10.11 (decided), Q10.12 (decided), Q10.13 (decided)
- **Proves it:** the smoke battery runs green on a clean checkout with one command.

### H2 — Index batteries
- **Builds:** the batteries named in B1–B9 (field-type matrix, representation cases, golden documents,
  membership and grant rules, import errors, rebuild counts, swap under load, schema upgrade), each
  file per task as [B-indexing.md](B-indexing.md) lists them, labels from the schema register.
- **Size:** S · **Depends on:** H1, B1–B9
- **Proves it:** the batteries named in area B.

### H3 — Search batteries
- **Builds:** one battery per D/E/F task (hook shapes, parse table, compiler, keywords, nodes,
  restrictions and the config matrix, access, group filter, `J`, sort on a ties fixture, windows,
  hydrate shapes, each mode, each field/date/number form), plus the factorial combinations battery
  (512 resources, nine two-valued attributes, expected sets).
- **Size:** S each · **Depends on:** H1 and the task
- **Must honour:** core's keyword-less relevance and the node-only order are expected differences,
  not failures, once Q7.1 is decided — the accepted-divergence list carries them.
- **Proves it:** the battery named in each task.

### H4 — Live API runner, user matrix and catalogue
- **Builds:** `harness/live/`: signed API calls as paired users (Typesense-only group / plugin-off
  group), credentials from environment variables only, a pause between calls, `--slow` for
  date-field searches; the catalogue cases and table generator; a read-only DB survey.
- **Size:** M · **Depends on:** H1
- **Must honour:**
  - The test host overheats under back-to-back heavy MySQL searches: no AND chains above ~40 nodes,
    no big `IN` lists, spacing between heavy cases, ask before minutes-long runs.
  - Core date-field searches take 6–8 s each there; the full catalogue is ~1 hour at 5 s pauses.
  - An accent and case case per field type runs live (`café` / `cafe`), because the local harness
    cannot check collation (Q2.18).
  - Credentials are pasted per session, never stored (Q10.5); the permission system blocks reading
    them from other sessions' files.
  - The user matrix: standard, `J` with `j`/`-j`, `J` with `j*`, `R`/`h`, `e2`, `ert`, a search
    filter, a custom grant — each as a pair (Q10.8).
- **Blocking questions:** Q10.4, Q10.5, Q10.8, Q10.9, Q9.9
- **Proves it:** a light live run after each milestone; the catalogue regenerated before a release.

### H5 — Freshness, outage and rebuild batteries
- **Builds:** the freshness cases of [C-event-register.md](C-event-register.md) (one per row, in the
  C5–C7 battery files) and the sync plan's Phase 3 cases: each event followed by the same search as both
  users; a bulk window; hook-less writes with raw SQL then a rebuild; stop/start Typesense with
  edits in between; edits during a build; the builder-path equality test.
- **Size:** M · **Depends on:** H1, C1–C8
- **Blocking questions:** Q10.6
- **Proves it:** all green locally; the freshness subset live.

### H6 — Performance checks
- **Builds:** latency samples core vs plugin for a fixed case list on the test host (lightly), the
  rebuild's timing report, and an edit-burst soak: a hundred API edits in a minute, the queue
  drained within the freshness target with no parked row, the worker's rate recorded.
- **Size:** S · **Depends on:** H4, B8
- **Blocking questions:** Q10.7
- **Proves it:** figures recorded in the plugin docs per release.

### H7 — Core-bug batteries
- **Builds:** keep the batteries that reproduce core's own bugs (Core 1–10, the CSV TypeError, the
  `!last` + node bucket row) so an upstream fix is noticed; the rows are the ones marked *bug* in
  `catalogue.md`.
- **Size:** S · **Depends on:** H1
- **Blocking questions:** Q10.10
- **Proves it:** they run as "expected to fail" and flip when core changes.

### H8 — The second dataset and the live user matrix
- **Builds:** a seeded test system (the test-system seeder) holding what the YSP database lacks — a
  category tree in use, HTML and translated values, a complete-indexed field, a non-indexed date
  and a non-indexed numeric field, related keywords, search filters, custom grants with expiry,
  featured, request, research and smart collections, users with `J` and `j` / `-j` / `j*`, `R` and
  `h` users, `e2`, `ert`; the paired API users (a Typesense-only group and a plugin-off group) for
  each permission combination, created by script (Q10.8); the catalogue's example values
  regenerated for it (`catalogue_values.php`); a second live catalogue run.
- **Size:** M · **Depends on:** H4, B8 (an index of it)
- **Must honour:** principle 11 — parity is measured on two differently shaped datasets, not one;
  the seeder produces non-customer data; the new users' credentials never enter the repo (Q10.5);
  the host's thermal limit applies to its live runs too.
- **Blocking questions:** Q10.4, Q10.8, Q10.5
- **Proves it:** the catalogue run on the second system agrees with core for every `must` row the
  plugin serves on YSP, with the same accepted-divergence list.

### H9 — MySQL-backed harness mode
- **Builds:** a second database driver in `harness/ab/boot.php` (`HARNESS_DB=mysql`): core's SQL runs
  unchanged against a local MySQL (a container — the machine has no MySQL) loaded from `dbstruct/`
  with the same fixture; the same batteries, runner and summariser; the CI job runs it if Q10.2 says
  so.
- **Size:** M · **Depends on:** H1
- **Must honour:** the SQLite mode stays the default for speed (thirty batteries in about twelve
  seconds); the MySQL mode is the proof for the forms the shim emulates — wildcards, the full-text
  box, phrases, accents, collation; MySQL's full-text settings (`innodb_ft_min_token_size`, the
  InnoDB stop-word list) must match the test host's, or the emulation is replaced by a different
  divergence; a fixture word must exist in the keyword table in either mode.
- **Blocking questions:** Q10.2
- **Proves it:** the emulated batteries agree with the MySQL mode on the fixture; the live catalogue
  rows for those forms agree with it too.
