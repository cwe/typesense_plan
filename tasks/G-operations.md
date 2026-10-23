# G — Operations

Milestone M7, except that G4's operations page starts in M1 (the rebuild) and grows.

### G1 — Setup page
- **Builds:** `pages/setup.php` — the admin-facing options (Q9.1 decided 2026-10-21: host, port,
  protocol, the two keys as password fields shown set / not set, enabled, the badge, the excluded
  fields — each drawn locked when `$typesense_search_<option>_override` is set in `config.php`, keys
  as hidden), the prefix read-only with the lockable "regenerate" action (Q9.12), health (ready flag,
  server version against the tested range, live generation and size, last rebuild time/result/
  duration, queue depth and oldest row, count drift vs MySQL, validation drops, breaker state, the
  rebuild schedule, the "rebuild required" notice), warnings (Typesense-only on for any group),
  "Rebuild now" and "Resync now" (the offline job, P11), both lockable.
- **Also builds (2026-10-20):** the excluded-fields control (G9).
- **Size:** M · **Depends on:** A1, A6, B8, C4
- **Must honour:** the rebuild schedule is the administrator's scheduled-task entry; the plugin
  reports it (P15); group config overrides are the A/B mechanism and must not be hidden by the page.
- **Blocking questions:** Q9.1, Q9.3, Q9.6, Q8.12, Q12.1
- **Proves it:** manual check on the test system; each health figure matches a direct query.

### G2 — Logging
- **Builds:** plugin `debug()` lines for every veto reason (through core's debug log, so off unless
  `$debug_log`); a dedicated sync/rebuild log holding refs and outcomes, never search strings or
  keys, with a configurable retention, thirty days by default (Q12.5, 2026-10-22).
- **Size:** S · **Depends on:** C4, B8
- **Must honour:** the harness reads the veto reason from the debug output — keep it one line per
  search; that line carries the search string and the parameters and exists only behind
  `$debug_log`, under core's existing policy (Q12.7, 2026-10-22); keys and request headers never
  appear in any line.
- **Blocking questions:** Q9.5
- **Proves it:** a failed rebuild and a parked queue row are diagnosable from the log alone.

### G3 — Indicator badge
- **Builds:** the search-title badge: served / fallback / index stale / bulk pending.
- **Size:** S · **Depends on:** D1
- **Must honour:** set only by the main grid search (D1).
- **Blocking questions:** Q9.6
- **Proves it:** manual check in each state.

### G4 — Documentation
- **Must say (Q7.1, 2026-10-20):** a search with keywords ordered by relevance is ordered differently
  from core — core scores by the matched nodes' hit counts, the plugin by Typesense's text match —
  while every other order is identical; the one documented difference.
- **Builds:** in the plugin: an operations page (install, configure, rebuild, monitor, recover), the
  schema document (every field, type, source, flags, reason), `CORE_PATCHES.md`, and updates to the
  behaviour spec and architecture doc as tasks land; in this repo: the decisions.
- **Size:** M (spread across milestones) · **Depends on:** —
- **Blocking questions:** Q0.9
- **Proves it:** a new administrator can install, configure and rebuild from the page alone.

### G5 — Deployment procedure and versioning
- **Builds:** the written procedure (deploy code, then rebuild; a schema change is a generation
  rebuild; the plugin yaml version bump rule; a migration note per schema version).
- **Decided 2026-10-20 (Q11.6):** the plugin ships inside ResourceSpace 11.1, so its version is
  ResourceSpace's and deployment follows the ResourceSpace release process; what remains here is the
  rebuild-on-upgrade rule and the migration note per schema version. **The GitHub branch is the
  working copy only (2026-10-23):** the release source is Montala's SVN trunk, into which Chris
  moves the code by hand and leaves `harness/` out; so nothing in the plugin may depend on the
  harness directory existing, and the core changes (I1, I3) travel in the same move.
- **Size:** S · **Depends on:** B9
- **Blocking questions:** Q9.7
- **Proves it:** one deployment to the test system follows it without improvisation.

### G6 — Several installations on one Typesense server
- **Builds:** the operations page section for a shared server: one prefix and one key pair per
  installation (A8, X1), memory sizing per tenant (about 800 MB a generation on the test index,
  two during a rebuild), staggered rebuild windows, the import-concurrency default for a busy
  server, the filter-operations limit shared by all, what each tenant's setup page can and cannot
  show (Q12.11); and the live proof on the test server.
- **Size:** S · **Depends on:** A8, B8, X1, G8
- **Must honour:** Q9.8 is decided — a requirement, not an option; the checklist is written for an
  operator who runs several installations and may not administer any of them.
- **Blocking questions:** Q9.13, Q12.11
- **Proves it:** two installations rebuilt against the test server from the page alone, each
  searching its own index while the other rebuilds; neither tenant's key can read the other.

### G7 — Search explain tool
- **Builds:** `scripts/explain.php --user <ref> --search '<string>' [--restypes] [--order_by]
  [--archive] [--sort] [--fetchrows]` printing, for that user: the hook arguments as core passes
  them (core run up to the hook with a capture flag, or the harness's trace layer), the plugin's
  parse result, the chosen mode and the restrictions applied, the plan, the compiled Typesense
  parameters (and GET or POST), the veto reason if any, the request time, the first refs and the
  total, and core's SQL for the same search (`returnsql`); `--json` for the batteries; `--both`
  runs core and Typesense and diffs the ref sets.
- **Size:** M · **Depends on:** D1–D11, G2
- **Must honour:** runs as the given user (permissions, search filter, `J`) without a session —
  the harness already builds a user context from `$userpermissions`; read-only (no keyword-usage
  logging, no smart-collection refresh); the veto reason is the same line G2 logs; on the test
  host it reads credentials from the environment, never from arguments.
- **Blocking questions:** —
- **Proves it:** the search batteries assert plans through `--json`; the operations page has one
  worked example per catalogue group; a "why does this search differ" question is answered from
  its output alone.

### G8 — Typesense server configuration
- **Builds:** the server-side checklist in the operations page (G4): the version pinned to 30.x and
  the versions verified; `--filter-by-max-ops` above the plugin's guard (Q3.9); memory from the
  schema register's budget (about 800 MB a generation on the test index, two during a rebuild,
  Q1.11) multiplied by the number of installations on the server (G6); `--api-key` plus a scoped search-only key (Q9.2); `--data-dir` and backups by rebuild
  (Q8.11); the two cron entries (worker every minute, nightly rebuild); permissions for the CLI
  scripts; log rotation. And a setup-page check (G1) that reads `/debug` and `/health`, compares the
  server's version and `filter_by_max_ops` with the plugin's assumptions and warns on a mismatch.
- **Sizing guidance (Q1.11, 2026-10-20):** the server needs room for two generations of the index
  plus headroom; reference points from the measured systems (about 800 MB per 115 k resources with
  fragments on the test system; the 1.4 M-resource database measured in M1); the levers for a short
  server, in order: the exclusion list (G9), no uncapped `_w` copy, fragments only on partial-indexed
  fields, option names once in a joined collection (a parity trade, last).
- **Server version (Q1.9, 2026-10-20):** Typesense 30.0 or later; the tested range is listed here and
  updated when a new major is proved; the setup page shows the server's version against it.
- **Shared server (Q9.13, 2026-10-21):** stagger each installation's rebuild window; set the
  filter-operations limit once on the server and mirror it into every tenant's `config.php`; raise a
  tenant's import streams above the default of two only where the server has room; monitor server-wide
  load, memory and version outside ResourceSpace — no tenant holds a server-wide key.
- **Size:** S · **Depends on:** A2, G4; the check with G1 · the checklist before the first live
  rebuild (M1), the check with G1 (M7)
- **Must honour:** the guard (P34) counts operations as the server does, so a raised server limit
  must raise the guard, not be hard-coded; the test host runs RS and Typesense on one 16 GB
  machine; a server upgrade is a schema-compatibility question (B9) — record the verified versions.
- **Blocking questions:** Q3.9, Q12.1, Q12.2, Q8.1, Q0.5
- **Proves it:** the test host configured from the checklist alone; the setup page warns when
  `--filter-by-max-ops` is lowered below the guard.

### G9 — Field exclusion list
- **Builds:** `$typesense_search_excluded_fields` (field refs, empty by default) in `config.php`, set from
  a control on the setup page (G1: every active field with a checkbox, POST with a form token); B2's
  selector honours it — an excluded field gets no representation, its nodes stay out of `nodes[]` and
  `populated_field_ids[]`; the decline rule in the parse and the pipeline — a search that names an
  excluded field (`field:…`, `!hasdata<field>`, a node token of it) declines, and a wildcard or
  full-text search declines for a user who can see an active excluded field; a change to the list bumps
  the schema version (B9) and queues a rebuild (C7).
- **Size:** S · **Depends on:** A1, B2, B9, D4, G1, SA5, SA8 · **Milestone:** M7 (the control); the
  option and the selector rule from M4, so SK1's battery runs with and without an exclusion
- **Must honour:** wanted beyond size (Chris, 2026-10-20) — a field an installation does not want copied
  to Typesense (Q12.3) is the same mechanism; an excluded field is exactly option (a) of Q1.1, so its
  decline logic is the only place (a) survives and it stays confined to the list; "can see" is per user
  (`metadata_field_view_access`), so the rule runs in X4's battery.
- **Blocking questions:** Q1.1 (decided), Q12.3
- **Proves it:** `harness/ab/G9_excluded_fields.php` — with the fixture's field 96 excluded: its values
  are absent from the document; `field:word` on it declines; `word*` and the full-text box decline for a
  user who can see it and serve for one who cannot; the list change forces a rebuild; a GET to the
  setup action changes nothing.
