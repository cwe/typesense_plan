# Chunk 8 — Index sync and freshness

Status: planning. Depends on: 0, 1, 11 (core hooks). Feeds: 5, 6, 7, 9.

## Scope

Keeping the three Typesense collections in step with ResourceSpace: capturing every change, applying
it, rebuilding safely, behaving during outages, and what to do until it all works.

## What we know

- A full design exists (`typesense-incremental-sync-plan.md`, 855 lines) with thirteen decisions taken
  and two open; its Phase 0–3 plan and event inventory are the starting point.
- Today nothing reaches the index between full reindexes. The on-save path is broken: an unset
  collection-name global (requests go to `/collections//…`), the document passed as the wrong
  argument, the old document shape (fixing only the first two would strip each edited resource's
  nodes, access and field values), and more in the plan's §1.1. The node layer has no hooks at all,
  so the plugin's node-delete hooks never fire. The collection add/remove hooks fire **before** the
  write with named arguments. Custom-access writers via requests and collection email have no hook
  and no log. `resource.modified` moves only through `resource_log()`. Core updates hit counts by
  cron with no hook.
- A group-restricted plugin's hooks run only for those groups and not in cron; `hook("cron")` runs
  under a 30-minute limit; `job_queue_add()` exists for offline jobs.
- Typesense 30.2 facts the design rests on: aliases (a reference binds to the real collection at
  creation; moving only the resources alias breaks joins; joins can be addressed via an alias);
  `cascade_delete` removes membership and grant documents; delete-by-filter works on reference
  fields; `update` on a missing id fails per line inside an HTTP 200; schema PATCH on a live
  collection; stored-only fields export but cannot be filtered; buffer-table sync every 5–10 s;
  concurrent imports ≤ cores − 2; on 503 back off 10–60 s with jitter.
- Rebuild performance: fresh upsert 770–1,800 docs/s, re-upsert 540/s, partial update 650–780/s (as
  costly as a full write), 4 parallel ≈ 2×, 2,000-doc chunks no faster than 500, memberships
  3,300/s (7,800/s parallel); a full run just over 1 minute for ~115k + ~152k.
- The harness has a freshness battery (`08_index_freshness.php`) and the sync plan's Phase 3 lists
  the freshness, bulk-window, hook-less, outage and rebuild cases.

## Candidate work items (from the sync plan's phases)

- Phase 0: shared document builder (one function for the rebuild and the single-resource path);
  queue table + `mark_dirty` + request-end flush + `scripts/sync.php` worker; existing hooks switched
  to marking; guards (ready flag, breaker, separate timeouts); read-time validation; plugin made
  global with per-group enable; `reindex.php` as a generation build with alias swap and catch-up.
- Phase 1: bulk rows with cursors, the `bulk_pending` MySQL fallback, 503 handling, pipelined
  imports, the admin rebuild job, setup-page stats.
- Phase 2: the core hooks (one PR).
- Phase 3: verification cases and tuning.

## Questions to answer before coding

### Q8.1 — Re-confirm the thirteen taken decisions (P7–P17, P24)
In particular: nightly rebuild behind aliases (is nightly often enough; is a rebuild window
available); a queue table drained by a per-minute cron worker (is a system cron available on the
target, or only RS's own `hook("cron")`); searches served by MySQL while bulk rows are pending; the
plugin becoming global with per-group enable; no deferred bulk work. Any to revisit?

### Q8.2 — Validation strictness
Same decision as Q6.1 — answer once.

### Q8.3 — Group override ordering
Open decision 2: is a per-group `$typesense_search_enabled` override applied before the search hook
runs? This is a test, not a question — record the result.

### Q8.4 — Core hooks (Phase 2)
The list in the plan's §6.11: node-layer `after_resource_nodes_changed`, `after_set_node`,
before/after node delete, `after_update_resource_type`, `after_resource_custom_access_change`,
`after_put_resource_data`, collection bulk hooks, arguments on the field-config hooks. Confirm the
list, and the interim: hook-less writes wait for the nightly rebuild.

**Note (2026-10-20, from Q11.1):** the hooks are core changes Chris can make, so Phase 2 need not wait
for M7; whether the interim is built depends on the ResourceSpace version floor (Q0.5).

### Q8.5 — Phase 0 scope and order
Confirm: builder first; then queue + flush + worker; then hooks; then guards; validation in Phase 0
(Q6.10); then the generation rebuild. Anything to pull forward (e.g. the generation rebuild before
the queue, since it fixes freshness nightly on its own)?

### Q8.6 — Outage behaviour
Circuit breaker + fall back to MySQL (never empty results); the queue keeps accumulating; the worker
retries with backoff. Confirm, and confirm how Typesense-only mode behaves in an outage (Q0.6).

### Q8.7 — Freshness target
Own edits visible on the next page load (request-end flush); others' within seconds (worker); bulk
changes (node rename, field config) applied intraday with MySQL serving meanwhile; hook-less writes
nightly. Acceptable? Any flow that needs synchronous visibility (upload then collection view)?

### Q8.8 — Grants and memberships
Same queue, kinds `grants` and `collection`. Grant writers without hooks (requests, collection email)
and collection bulk writes without hooks wait for the nightly rebuild until Phase 2 — acceptable?

### Q8.9 — Hit-count freshness
If Q7.1 chooses B, `hit_count` is refreshed only by the rebuild (daily). Acceptable?
**Catalogue marks (2026-10-16):** the hit-count sorts are *must*. `hit_count` is written only by the
maintenance cron's copy from `new_hit_count`, with no hook, so the index is exact as of the last
rebuild — schedule the nightly build after that cron, or accept the day's drift. `user_rating` has
the `modifyuserrating` hook (register row E27).

### Q8.10 — Rebuild window and capacity
Time of day; duration budget; disk and memory for two generations briefly (2 × ~800 MB on the test
index); delete the old generation immediately (P7) — confirm; what happens if the build fails
(keep serving the old generation; alert).

### Q8.11 — Topology
Single Typesense node, no replication, backups by rebuild. In scope?

### Q8.12 — Observability
Setup page: queue depth, oldest row age, last rebuild time/result/duration, failures, validation
drops. Alerts (email on failed rebuild)? What is wanted?

### Q8.13 — Which register rows must be immediate in M3?
[tasks/C-event-register.md](../tasks/C-event-register.md) marks every write with a hook (immediate
after C5–C7) or an interim ("nightly rebuild", with read-time validation hiding stale access and
state changes). The gap rows are the hook-less writers: the API's `add_resource_nodes`, csv fixed-list
columns, metadata templates, request approvals and declines, collection reorders and bulk changes,
node renames. Is nightly acceptable for all of them until I3/C9 land, or is one of them common
enough on the target system to pull the core hooks forward (Q8.4)?

## Done when

Phase 0 outcome from the sync plan: edits, uploads, state/access changes, deletes, grant edits and
single collection changes reach the index before the next page load or within seconds; stale rows
cannot leak; an outage falls back; the nightly rebuild corrects everything else; the Phase 3 cases
pass in the harness.
