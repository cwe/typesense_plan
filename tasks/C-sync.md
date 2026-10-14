# C — Sync

Keeping the index in step with MySQL between rebuilds. Nine tasks: C1 and C8 are milestone M2 (the
queue table is tiny and read-time validation is the only protection until the rest works), C2–C7
are M3, C9 is M7 (it needs the core hooks of I3). The map of every write that changes what a
search returns — its hook or its gap, its queue row, the package that handles it and the freshness
case that proves it — is [C-event-register.md](C-event-register.md); it plays the role the catalogue
plays for the search packages.

The design is the sync plan's (`typesense-incremental-sync-plan.md` on `typesense`, decisions
P7–P17, P24), re-confirmed by Q8.1: hooks only mark rows in a queue table; the request applies its
own rows before responding; a per-minute worker applies the rest; a nightly generation rebuild
corrects everything the hooks miss; bulk work is served by MySQL while it is pending.

**The contract with the other areas.** Every row of the event register has either a hook (and a
handler in C5, C6 or C9) or an interim written next to it ("nightly rebuild"). A search package
whose representation reads a table other than `resource`, `resource_node` or `resource_data` adds
its event to the register before it closes (B-indexing, contract rule 3).

### C1 — Queue table and mark-dirty
- **Covers:** the *queue row* column of every register row.
- **Builds:** `dbstruct/table_typesense_search_sync_queue.txt` (`ref`, `kind`, `object_ref`,
  `object_ref2`, `priority`, `queued`, `not_before`, `attempts`, `last_error`, `claimed`,
  `claimed_by`, `applied_generation`, `cursor`; UNIQUE (`kind`, `object_ref`, `object_ref2`);
  indexes on (`priority`, `queued`) and `claimed`); `include/sync.php` — `mark_dirty(kind, ref,
  ref2 = 0, priority)` with coalescing that keeps the most urgent priority, `claim(kinds, n,
  owner)`, `release()`, `stamp(generation)`, `park()`; the kinds `resource`, `grants`, `membership`,
  `memberships_of_resource`, `collection`, `node`, `field`, `field_deleted`, `full`; priorities 0
  visibility, 1 edit, 2 repair, 3 bulk, 9 parked.
- **Size:** S · **Depends on:** A1 · **Milestone:** M2
- **Must honour:** `check_db_structs()` creates the table for an active plugin; a claim older than
  the worker's lifetime is abandoned; no deferred priority and no ageing (P8, P24); rows are
  stamped with the generation they were applied to during a build instead of being deleted (P9).
- **Fixture:** none (the harness's SQLite creates the table from `dbstruct/`).
- **Battery:** `harness/ab/C1_queue.php` — insert, coalesce (3 then 1 → one row at 1; 1 then 3 →
  stays 1), claim and release, stale claim reclaimed, stamp, park, the unique key.
- **Blocking questions:** Q8.1, Q8.5

### C2 — Applying a row
- **Covers:** the *applied as* column of every register row.
- **Builds:** `apply_rows(rows, generation)` — in kind order `node/field → resource → grants →
  memberships_of_resource → membership → collection`: `resource R` → B4's document upserted, or a
  delete by `ref:=[…]` when MySQL no longer has it (`cascade_delete` removes memberships and
  grants); `grants R` → delete by filter then B6's documents; `membership (C, R)` → read the row
  **now** and upsert or delete `C:R`; `memberships_of_resource R` → delete by filter; `collection
  C` → B5's documents for the collection replaced; bulk parents are C7's; every kind idempotent;
  rows deleted on success or stamped during a build; failures recorded with `attempts`,
  `not_before` and `last_error`.
- **Size:** M · **Depends on:** C1, B4, B5, B6, B7 · **Milestone:** M3
- **Must honour:** a resource in the deletion state keeps its document with its archive value
  (P14) and loses its memberships when `$remove_deleted_resources_from_collections`; a hard delete
  removes the document (the ref is captured in the *before* hook because the after-hook has no
  arguments); re-adding a resource to a collection resets `date_added` and NULLs `sortorder`;
  `collection_resource` may hold duplicate rows — the document id de-duplicates them; the same
  apply code serves the flush, the worker and the rebuild's catch-up.
- **Fixture:** the base fixture.
- **Battery:** `harness/ab/C2_apply.php` — for every kind: apply on the fixture and compare with a
  fresh rebuild's document; the delete paths; soft delete; re-add; a stamped row during a
  simulated build.
- **Blocking questions:** Q8.1

### C3 — Request-end flush
- **Covers:** the "next page load" freshness target (Q8.7).
- **Builds:** a shutdown function registered by the first `mark_dirty()` of a request: claims the
  request's own rows, applies them with C2 in kind order **before** the response is sent (no
  `fastcgi_finish_request`), bounded by `$typesense_search_flush_max_rows`; skipped when the
  breaker is open, `ready` is false, the process is the worker or the rebuild, or the request is
  CLI; a failed flush opens the breaker and leaves the rows.
- **Size:** S · **Depends on:** C1, C2 · **Milestone:** M3
- **Must honour:** a redirect to the search page after a save must already see the change; the
  cost is a few milliseconds per save; over-cap requests leave their rows and set `bulk_pending`.
- **Fixture:** the base fixture; the harness ends a "request" by calling the shutdown function.
- **Battery:** `harness/ab/C3_flush.php` — `update_field()` then the search in the next request is
  identical for both users; an over-cap batch leaves rows; breaker open → rows left, nothing else
  happens; a CLI process does not flush.
- **Blocking questions:** Q8.7

### C4 — Worker
- **Covers:** the "within seconds" and outage rows of the register.
- **Builds:** `scripts/sync.php` — the process lock with stale detection (not the four-hour
  `$process_locks_max_seconds`); health (each alias present, the generation matches the state; a
  missing alias → `ready = false` and an admin notification); breaker probe; a ~55 s loop claiming
  by priority with a reserved share for bulk, applying with C2; retries at 1, 5, 15, 60 minutes then
  priority 9; 503 → pause bulk work and back off 10–60 s with jitter, no attempt counted; idle poll
  5 s; `bulk_pending` set when any priority-3 row exists; `last_worker_run` and counters; the cron
  entry documented (and the `hook("cron")` fallback if Q8.1 says no system cron).
- **Size:** M · **Depends on:** C1, C2, A2, A6 · **Milestone:** M3
- **Must honour:** `hook("cron")` runs under a 30-minute limit and only with the plugin global
  (P13); several workers may run with disjoint claims through `claimed_by`; speed-ups (overlapped
  building and uploading, chunk size) are applied after measuring, never before (P16).
- **Fixture:** the base fixture; the harness starts and stops its private Typesense.
- **Battery:** `harness/ab/C4_worker.php` — a seeded queue drains in priority then FIFO order; a
  failing row backs off then parks; a 503 pauses without counting; stop Typesense, queue edits,
  restart, drain → identical results and searches fell back meanwhile; the stale lock.
- **Blocking questions:** Q8.1, Q8.6, Q9.5

### C5 — Resource events through existing hooks
- **Covers:** register rows E1–E12 (resource lifecycle and metadata writes that have a hook).
- **Builds:** handlers in `hooks/all.php` that only call `mark_dirty()`, with the exact parameter
  names: `resourcecreate`, `afternewresource` (also `grants R` — copies carry grants),
  `afterpluploadfile`, `uploadfilesuccess` (`$resource_ref`), `afterpreviewcreation`,
  `after_update_resource` (`$resourceId`), `update_field`, `aftersaveresourcedata` (first argument
  an int or an array; also `grants R…` at priority 0), `saveextraresourcedata($list)`,
  `after_update_archive_status($resources[], …)` (priority 0; plus `memberships_of_resource` when
  the state is the deletion state and `$remove_deleted_resources_from_collections`),
  `beforedeleteresourcefromdb($ref)` (priority 0; resolved as a delete by C2), `afterreplacefile`
  and `after_alt_upload` (once SF9's file fields exist).
- **Size:** M · **Depends on:** C1, C2, C3 · **Milestone:** M3
- **Must honour:** `hook()` passes string keys as named arguments on PHP 8.2 — a handler's
  parameter names must match the sync plan's Appendix B; one upload fires ~10 hooks and many
  `update_field` hooks: the queue coalesces them; `after_update_archive_status` fires even when the
  state did not change; the early returns in `save_resource_data` (locked, field errors) and
  `save_resource_data_multi` skip the after-hooks after some writes — register gaps corrected by
  the rebuild; `resourcecreate` fires before the metadata is written (the later hooks carry it).
- **Fixture:** the base fixture; cases call the real core functions (`create_resource`,
  `copy_resource`, `update_field`, `update_archive_status`, `delete_resource`, `update_resource`).
- **Battery:** `harness/ab/C5_resource_events.php` — one case per register row, each the core
  write followed by a flush and the same search as both users; a resource with ten hooks in one
  request produces one row.
- **Blocking questions:** Q8.4, Q8.13 (new)

### C6 — Collection and grant events through existing hooks
- **Covers:** register rows E30–E33 and E40–E42 (the collection and grant writers with a hook) and
  the gap rows' interim.
- **Builds:** `Addtocollectionsuccess` / `Removefromcollectionsuccess` handlers declared exactly
  `($resourceId, $collectionId)` — they fire **before** the write, so the row is applied after it
  by the flush or the worker; `membership (C, R)` at priority 1; grants from
  `aftersaveresourcedata` (it cannot tell whether grants changed, so `grants R` is always queued at
  priority 0) and `afternewresource`; `additional_email_resource` for group grants via email; the
  hook-less writers (reorder, add all results, empty, copy with wipe, delete collection, type /
  public / user changes, clean-up, request approval and decline, the share page's delete, csv
  grants) documented as waiting for the nightly rebuild until C9.
- **Size:** S · **Depends on:** C1, C2, C3, B5, B6 · **Milestone:** M3
- **Must honour:** `Removefromcollectionsuccess` fires even when the resource is not a member; the
  add is always delete + re-insert; smart collections go through the same per-resource hooks;
  selection collections are not indexed (P27), so their writes (including the logout clear) need
  no row; leaving access 3 deletes group grant rows and fires `aftersaveresourcedata`.
- **Fixture:** the base fixture's collections and grants.
- **Battery:** `harness/ab/C6_collection_grant_events.php` — add, remove, re-add (order and
  `date_added` as core), a group grant, a user grant with expiry, leaving access 3; each followed by
  the search as both users; a reorder with raw SQL stays stale until a rebuild (the documented
  interim).
- **Blocking questions:** Q8.4, Q8.8, Q5.6

### C7 — Bulk and configuration events
- **Covers:** register rows E20–E27 (node definitions, field config, global config, related
  keywords, hit counts).
- **Builds:** parent rows with a `cursor` for `node`, `field`, `field_deleted`, `collection` and
  `full`, streamed through C2 in ref order; the field-config snapshot (the index-relevant columns
  of every field, stored by the rebuild) diffed on `afterresourcetypefieldeditsave` → `field F`
  when a flag, type, constraint or active state changed; `after_delete_resource_type_field` →
  `field_deleted F` (export `populated_field_ids:=F` from Typesense for the refs); a global-config
  snapshot (`$stemming`, `$view_title_field`, `$date_field`, `$config_separators`,
  `$partial_index_min_word_length`) diffed by the worker → "rebuild required" (B9); the `node`
  kind reachable from the setup page now and from C9's hooks later; `after_save_related_keywords`
  → the synonym set (if SA10 chooses synonyms); the `bulk_pending` flag and the serving rule D1
  reads (MySQL serves while it is set, P10); the reserved bulk share.
- **Size:** M · **Depends on:** C4, B4, B9, A6 · **Milestone:** M3
- **Must honour:** a node rename touches every resource using the node (thousands) — core itself
  only remaps `node_keyword`; a numeric-flag change moves a field between `_s` and `_f`+`_q` and
  changes its `query_by` mapping; `$stemming` is a rebuild, never a resync; the field hooks have
  no arguments (`$GLOBALS['ref']`, page context only); hit counts are updated by core's cron with
  no hook, so `hit_count` (if Q7.1 chooses B) is fresh only after a rebuild (Q8.9).
- **Fixture:** the base fixture plus a generated field with values on many resources.
- **Battery:** `harness/ab/C7_bulk.php` — flip a field's index flag → every resource with values
  is streamed and the search is served by MySQL until done, then identical from Typesense; delete a
  field; a `node` parent row for a renamed node; a config snapshot change sets "rebuild required";
  the bulk share keeps edits flowing during a bulk run.
- **Blocking questions:** Q8.1, Q8.7, Q2.10

### C8 — Read-time validation and repair
- **Covers:** the stale-row protection for every register gap until its hook exists.
- **Builds:** `include/search/validate.php`, called from hydrate (D11): the permission part of the
  filter rebuilt on its own (`search_filter()` plus the `rca`/`rca2` joins) and applied in the
  hydrate query; a dropped ref → option B (fall back for the whole search, Q6.1), a priority-2
  `resource R` row, a counter the setup page shows.
- **Size:** M · **Depends on:** C1, D11 · **Milestone:** M2
- **Must honour:** the hook's `$sql_filter` already contains keyword criteria, so the permission
  part is rebuilt separately; refs-only hydrate must add the grant joins too; veto when
  `$heightmin` is set unless replicated; the `J` restriction cannot be re-checked without the
  collection join; stale rows expose existence, title and preview, not the file; the window is the
  rebuild interval until C3–C7 land.
- **Fixture:** the base fixture.
- **Battery:** `harness/ab/C8_validation.php` — change access, workflow state, a grant, a filter
  node, a type, and delete, each searched before any sync runs → the fallback, never the resource;
  a repair row queued each time.
- **Blocking questions:** Q6.1, Q6.2, Q6.10

### C9 — Handlers for the core hooks (after I3)
- **Covers:** the register's gap rows: node writers, type change, `put_resource_data`, the grant
  writers, the collection bulk writers, the related-resources writers, the lock.
- **Builds:** handlers for `after_resource_nodes_changed($resources[])` → `resource R…` (1);
  `after_set_node($node, …)` → `node N` (3); `beforenodedelete` (capture the refs) /
  `afternodedelete` → `node N` (3); `after_update_resource_type` → `resource R` (0);
  `after_resource_custom_access_change` → `grants R` (0); `after_put_resource_data` → `resource R`
  (0); the collection bulk hooks → `collection C` (3); `after_related_resources_changed` →
  `resource R…` (1, when SF11 exists); `after_resource_lock_change` → `resource R` (1, when SF13
  exists); the field-config hooks with arguments replacing C7's snapshot diff.
- **Size:** S · **Depends on:** I3, C5, C6, C7 · **Milestone:** M7
- **Must honour:** the node-layer hook fires inside the save transaction — handlers only mark and
  act at request end; plain positional arguments, no string keys.
- **Battery:** the gap rows' cases in `C5`–`C7`'s batteries flip from "stale until the rebuild" to
  "identical after the flush".
- **Blocking questions:** Q8.4, Q11.1
