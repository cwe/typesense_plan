# G — Operations

Milestone M7, except that G4's operations page starts in M1 (the rebuild) and grows.

### G1 — Setup page
- **Builds:** `pages/setup.php` — the admin-facing options (Q9.1), health (ready flag, last rebuild
  time/result/duration, queue depth and oldest row, count drift vs MySQL, validation drops),
  warnings, "Rebuild now" and "Resync now" (the offline job, P11), the Typesense-only warning text.
- **Size:** M · **Depends on:** A1, A6, B8, C4
- **Must honour:** the rebuild schedule is the administrator's scheduled-task entry; the plugin
  reports it (P15); group config overrides are the A/B mechanism and must not be hidden by the page.
- **Blocking questions:** Q9.1, Q9.3, Q9.6
- **Proves it:** manual check on the test system; each health figure matches a direct query.

### G2 — Logging
- **Builds:** plugin `debug()` lines for every veto reason; a dedicated sync/rebuild log with
  retention.
- **Size:** S · **Depends on:** C4, B8
- **Must honour:** the harness reads the veto reason from the debug output — keep it one line per
  search.
- **Blocking questions:** Q9.5
- **Proves it:** a failed rebuild and a parked queue row are diagnosable from the log alone.

### G3 — Indicator badge
- **Builds:** the search-title badge: served / fallback / index stale / bulk pending.
- **Size:** S · **Depends on:** D1
- **Must honour:** set only by the main grid search (D1).
- **Blocking questions:** Q9.6
- **Proves it:** manual check in each state.

### G4 — Documentation
- **Builds:** in the plugin: an operations page (install, configure, rebuild, monitor, recover), the
  schema document (every field, type, source, flags, reason), `CORE_PATCHES.md`, and updates to the
  behaviour spec and architecture doc as tasks land; in this repo: the decisions.
- **Size:** M (spread across milestones) · **Depends on:** —
- **Blocking questions:** Q0.9
- **Proves it:** a new administrator can install, configure and rebuild from the page alone.

### G5 — Deployment procedure and versioning
- **Builds:** the written procedure (deploy code, then rebuild; a schema change is a generation
  rebuild; the plugin yaml version bump rule; a migration note per schema version).
- **Size:** S · **Depends on:** B9
- **Blocking questions:** Q9.7
- **Proves it:** one deployment to the test system follows it without improvisation.

### G6 — Several instances on one Typesense server
- **Builds:** confirmation that the prefix isolates instances end to end (aliases, generations, queue).
- **Size:** S · **Depends on:** B8
- **Blocking questions:** Q9.8
- **Proves it:** two prefixes on the local Typesense rebuilt and searched independently.

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
  Q1.11); `--api-key` plus a scoped search-only key (Q9.2); `--data-dir` and backups by rebuild
  (Q8.11); the two cron entries (worker every minute, nightly rebuild); permissions for the CLI
  scripts; log rotation. And a setup-page check (G1) that reads `/debug` and `/health`, compares the
  server's version and `filter_by_max_ops` with the plugin's assumptions and warns on a mismatch.
- **Size:** S · **Depends on:** A2, G4; the check with G1 · the checklist before the first live
  rebuild (M1), the check with G1 (M7)
- **Must honour:** the guard (P34) counts operations as the server does, so a raised server limit
  must raise the guard, not be hard-coded; the test host runs RS and Typesense on one 16 GB
  machine; a server upgrade is a schema-compatibility question (B9) — record the verified versions.
- **Blocking questions:** Q3.9, Q9.2, Q8.1, Q0.5
- **Proves it:** the test host configured from the checklist alone; the setup page warns when
  `--filter-by-max-ops` is lowered below the guard.
