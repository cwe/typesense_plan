# Chunk 9 — Reindexer, configuration and operations

Status: planning. Depends on: 0, 8. Feeds: 10.

## Scope

`scripts/reindex.php` and its performance, the plugin's configuration options and setup page, where
secrets live, how a change is deployed, and how an administrator sees that the index is healthy.

## What we know

- `reindex.php`: one pass loop over resources, memberships, attributes and grants; progress and
  ETA; failure reasons; a summary table with Typesense vs MySQL counts; exit codes 0/1/2;
  `--drop` (no search fallback while rebuilding); per-line import error checks; 500-doc / 4 MB
  chunks; a keep-alive cURL handle; keyset pagination; grants batched by whole resource.
- Schema upgrades at reindex time: `PATCH` adds and drops fields with a 600 s timeout
  (`created_by` made optional by drop-and-re-add in one PATCH, values kept).
- Config: host, port, protocol, API key, collection prefix (repo default `test3_`), timeout,
  `$typesense_search_global_filter`, enabled, Typesense-only, indicator badge,
  `$typesense_search_max_rows` (25,000), `$typesense_search_filter_max_ops` (100), import max
  docs/bytes. Setup page: the server fields and three toggles plus max rows. Group config overrides
  are used on the test system to run A/B groups.
- Performance improvements listed and not yet done: one complete document per resource (drop the
  second write), build into an empty generation, 3–4 imports in flight via `curl_multi`, encode JSON
  once, bigger MySQL batches for the attributes pass, check with `EXPLAIN` whether the title derived
  table is materialised.
- Memory ~800 MB resident with fragments; the indexer host must run the new code before a reindex
  (a rebuild once ran old code); the test host overheats under heavy MySQL load.
- Deployment so far: Chris commits, deploys to the host, reindexes (`--drop` after a schema change).

## Candidate work items

- The generation rebuild (chunk 8) subsumes most of `reindex.php`; the performance list applies to it.
- A final config list with defaults and admin-facing subset; the setup page to match.
- A deployment and operations page in the plugin docs.
- Health and warnings on the setup page.

## Questions to answer before coding

### Q9.1 — Configuration
The final option list and defaults: collection prefix default (not `test3_`), separate search and
import timeouts, max rows, max filter ops, import concurrency and chunk sizes, worker count,
validation on/off, Typesense-only, rebuild schedule display. Which are admin-facing on the setup
page, which are `config.php` only?

### Q9.2 — Secrets
The API key is plugin config (stored in the database when set from the setup page). Acceptable?
Separate search-only and admin keys (the worker and rebuild need write access; search does not)?

### Q9.3 — Rebuild and resync from the UI
CLI only, or the admin "Rebuild now" offline job from the sync plan plus "Resync now" for a
resource/collection, with progress on the setup page?

### Q9.4 — Performance work: now or after sync
Apply the listed improvements as part of the generation rebuild, or first? What rebuild time is
acceptable on the target (the test system takes ~1 minute)?

### Q9.5 — Logging
Plugin `debug()` lines go to RS's debug log. A dedicated sync/rebuild log with retention? What
should an administrator be able to read after a failure?

### Q9.6 — Health, warnings and the badge
Ready flag semantics; what the indicator badge shows (served / fallback / index stale / bulk
pending); setup-page warnings (count drift vs MySQL, failed rebuild, queue backlog, validation
drops); any alerting.

### Q9.7 — Deployment procedure and versioning
Document: deploy code before reindex; schema change → generation rebuild; the plugin yaml version
and when it bumps; a migration note per schema version.

### Q9.8 — Several RS instances on one Typesense server
Prefix per instance is already supported. A requirement, or not?

### Q9.9 — Test hosts
The current host overheats under heavy MySQL load. Is a second or bigger test host planned, and is
the YSP database the reference dataset going forward?

## Done when

An operations page exists (install, configure, rebuild, monitor, recover); the setup page shows
health; a rebuild on the target meets the agreed time budget; secrets handling is decided and
documented.
