# H — Verification

Milestone M0 for H1, then one battery per task as the tasks land. The harness is the one piece of
existing code the rebuild is likely to keep (Q10.11).

### H1 — Harness foundation in the repo
- **Builds:** `plugins/typesense_search/harness/`: `boot.php` (SQLite from `dbstruct/`, the real
  core `*_functions.php` with the SQL translations, a private local Typesense), `fixture_lib.php`,
  the A/B runner (`do_search()` with the plugin off then on; ref sets, totals, order, row keys), the
  **accepted-divergence list** the runner reads, `run_all.sh`, `summarise.php`; optionally a CI job.
- **Size:** M · **Depends on:** —
- **Must honour:**
  - Not `tests/` — core's test runner scans `plugins/*/tests` (P39).
  - Gotchas: define `RS_TEST_MODE` or `metadata_field_view_access()` returns true on the CLI; core
    `exit()`s on an invalid search-filter id; a fixture word must exist in the keyword table or core
    returns before the hook; the veto reason is read from the debug file.
  - SQL translations: full text emulated (mark its results as emulated), `RLIKE` → `REGEXP`, `UNION`
    parentheses, `curdate()`, `IF` → `iif`, numeric compares `CAST AS REAL`, `ORDER BY ref DESC
    LIMIT` → `r.ref`, `get_nodes()` answered directly, `COLLATE NOCASE` for `_ci`; accents are not
    emulated.
  - A fact a case reports as "same" must be checked for the option really being applied (a runner
    once merged options with `array_merge` and six cases were wrongly recorded as matched).
- **Blocking questions:** Q10.1, Q10.2, Q10.3, Q10.11
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
  - Credentials are pasted per session, never stored (Q10.5); the permission system blocks reading
    them from other sessions' files.
  - The user matrix: standard, `J` with `j`/`-j`, `J` with `j*`, `R`/`h`, `e2`, `ert`, a search
    filter, a custom grant — each as a pair (Q10.8).
- **Blocking questions:** Q10.4, Q10.5, Q10.8, Q10.9
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
- **Builds:** latency samples core vs plugin for a fixed case list on the test host (lightly), and the
  rebuild's timing report.
- **Size:** S · **Depends on:** H4, B8
- **Blocking questions:** Q10.7
- **Proves it:** figures recorded in the plugin docs per release.

### H7 — Core-bug batteries
- **Builds:** keep the batteries that reproduce core's own bugs (Core 1–10, the CSV TypeError, the
  `!last` + node bucket row) so an upstream fix is noticed.
- **Size:** S · **Depends on:** H1
- **Blocking questions:** Q10.10
- **Proves it:** they run as "expected to fail" and flip when core changes.
