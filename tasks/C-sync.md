# C — Sync

Milestone M3 (C7 is built in M2 because it is the only protection until the rest works). The design is
the sync plan's Phase 0–2; these tasks are its pieces.

### C1 — Queue table and mark-dirty
- **Builds:** `dbstruct/table_typesense_search_sync_queue.txt`; `include/sync.php` —
  `mark_dirty(kind, ref, priority)`.
- **Size:** S · **Depends on:** A1
- **Must honour:**
  - Unique on `kind + ref`, lowest priority wins; kinds `resource`, `collection`, `grants`, `node`,
    `field`, `field_deleted`, `full`; priorities 0 visibility, 1 edit, 2 repair, 3 bulk, 9 parked;
    no deferred priority (P8, P24).
  - Rows are stamped with the generation they were applied to (P9).
- **Blocking questions:** Q8.1, Q8.5
- **Proves it:** unit tests for insert/upsert/priority/stamp.

### C2 — Request-end flush
- **Builds:** a shutdown function that applies the request's own rows before the response.
- **Size:** S · **Depends on:** C1, B2–B4, A2
- **Must honour:**
  - Before the response is sent, no `fastcgi_finish_request`, a short write timeout of its own;
    anything it cannot apply is left for the worker (P8 decision 8).
  - Own edits visible on the next page load is the freshness target (Q8.7).
- **Blocking questions:** Q8.7
- **Proves it:** edit a field via `update_field`, search in the next request, identical to core.

### C3 — Worker
- **Builds:** `scripts/sync.php` — a 55 s loop polling every 5 s, priorities, retries with backoff,
  parking, a lock; the cron entry documented.
- **Size:** M · **Depends on:** C1, A2
- **Must honour:**
  - `hook("cron")` runs under a 30-minute limit and only with the plugin global (P13); a system cron
    per minute is the design — whether one is available on the target is Q8.1.
  - On 503 halve concurrency and back off; on repeated failure park the row and count it.
- **Blocking questions:** Q8.1, Q9.5
- **Proves it:** the worker drains a seeded queue; an outage test (stop Typesense, queue, restart,
  drain) passes.

### C4 — Event capture through existing hooks
- **Builds:** `hooks/all.php` handlers that only call `mark_dirty`.
- **Size:** M · **Depends on:** C1
- **Must honour (sync plan Appendix B):**
  - `aftersaveresourcedata` receives an int or an array; `update_field`; `resourcecreate`,
    `afternewresource`, `afterpluploadfile`, `afterpreviewcreation`, `after_update_resource`,
    `after_update_archive_status` (moving to the deletion state removes memberships),
    `beforedeleteresourcefromdb` (delete the document; `cascade_delete` removes memberships and
    grants).
  - `Addtocollectionsuccess` / `Removefromcollectionsuccess` fire **before** the write with named
    arguments: the handler must be declared exactly `($resourceId, $collectionId)` (PHP 8.2 passes
    the keys as named arguments), and the row is applied after the write (the flush), not inside
    the hook.
  - Grants: `aftersaveresourcedata`; grant writers via requests and collection email have no hook
    → nightly rebuild until I3.
  - The node layer has zero hooks: do not register node-delete handlers that never fire; they return
    with I3.
  - `after_save_related_keywords` → synonym set update (if Q2.10 chooses synonyms).
- **Blocking questions:** Q8.4, Q2.10
- **Proves it:** a freshness battery with one case per event, each followed by the same search as
  both users.

### C5 — Bulk kinds and the serving rule
- **Builds:** parent rows with a cursor for `node`, `field`, `field_deleted`, `collection`, `full`;
  the field-config snapshot diff; the `bulk_pending` flag; searches served by MySQL while it is set.
- **Size:** M · **Depends on:** C3, B2
- **Must honour:**
  - All bulk work intraday, MySQL serving meanwhile, imports pipelined and parallel within the cap
    (P10, P24).
  - A node rename touches every resource using the node (thousands); a numeric-flag change moves a
    field between `_s` and `_f`+`_q` and changes its `query_by` mapping; `$stemming` is a rebuild.
- **Blocking questions:** Q8.1, Q8.7
- **Proves it:** rename a node used by thousands of resources → searches served by MySQL until the
  worker finishes, then identical from Typesense.

### C6 — Core hook handlers (Phase 2)
- **Builds:** handlers for the hooks added by I3, marking `node`, `resource`, `grants`, `collection`.
- **Size:** S · **Depends on:** I3, C4
- **Blocking questions:** Q8.4, Q11.1
- **Proves it:** the hook-less cases of the freshness battery now pass without a rebuild.

### C7 — Read-time validation and repair
- **Builds:** `include/search/validate.php`, called from hydrate (D11).
- **Size:** M · **Depends on:** C1, D11
- **Must honour:**
  - The hook's `$sql_filter` already contains keyword criteria, so the permission part is rebuilt
    on its own: `search_filter()` plus the `rca`/`rca2` joins, applied in the hydrate query.
  - Refs-only hydrate must add the grant joins too; veto when `$heightmin` is set unless replicated;
    the `J` restriction cannot be re-checked without the collection join.
  - A dropped row means the index is stale for that search: option B (fall back for the whole search,
    queue a repair, count it) was recommended over dropping and keeping the total (Q6.1).
  - Stale rows expose existence, title and preview, not the file; the window today is the full
    reindex interval.
- **Blocking questions:** Q6.1, Q6.2, Q6.10
- **Proves it:** the stale-index battery: change access, workflow state, a grant, a filter node, a
  type, and delete, each searched before the sync runs → the fallback, never the resource.
