# G — Operations

Milestone M7, except that G4's operations page starts in M1 (the rebuild) and grows.

### G1 — Setup page
- **Builds:** `pages/setup.php` — the admin-facing options (Q9.1), health (ready flag, last rebuild
  time/result/duration, queue depth and oldest row, count drift vs MySQL, validation drops),
  warnings, "Rebuild now" and "Resync now" (the offline job, P11), the Typesense-only warning text.
- **Size:** M · **Depends on:** A1, B5, C3
- **Must honour:** the rebuild schedule is the administrator's scheduled-task entry; the plugin
  reports it (P15); group config overrides are the A/B mechanism and must not be hidden by the page.
- **Blocking questions:** Q9.1, Q9.3, Q9.6
- **Proves it:** manual check on the test system; each health figure matches a direct query.

### G2 — Logging
- **Builds:** plugin `debug()` lines for every veto reason; a dedicated sync/rebuild log with
  retention.
- **Size:** S · **Depends on:** C3, B5
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
- **Size:** S · **Depends on:** B6
- **Blocking questions:** Q9.7
- **Proves it:** one deployment to the test system follows it without improvisation.

### G6 — Several instances on one Typesense server
- **Builds:** confirmation that the prefix isolates instances end to end (aliases, generations, queue).
- **Size:** S · **Depends on:** B5
- **Blocking questions:** Q9.8
- **Proves it:** two prefixes on the local Typesense rebuilt and searched independently.
