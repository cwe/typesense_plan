# B — Indexing

Milestone M1. One builder per collection, used by both the rebuild and the sync path, so a document
is the same whichever path wrote it.

### B1 — Field inventory and representation
- **Builds:** `include/fields.php` — `indexed_fields()`, `representation(field)` → suffixes, type,
  searchable, sortable, flags.
- **Size:** S · **Depends on:** A3
- **Must honour:**
  - Numeric single-line field = `field_constraint == 1`, value indexed as `_f` only when
    `is_numeric` (the old `$value == 1` bug); the search UI and core's `+0` sort key off the same
    flag.
  - `partial_index` fields get `_p` fragments; the lookup must *return* the flag (an earlier fix was
    inert because the lookup selected it but did not return it — test through the lookup).
  - Inactive fields excluded from positive matching and from attributes (`rtf.active = 1`).
  - Which fields contribute values depends on Q1.1 (flagged only, or all); `query_by` always
    excludes non-indexed fields for plain keyword searches.
- **Blocking questions:** Q1.1, Q1.2, Q1.3
- **Proves it:** a table test over every field type × flag combination.

### B2 — Resource document builder
- **Builds:** `include/documents.php` — `build_resource_documents(refs[])`.
- **Size:** M · **Depends on:** A4, A5, B1
- **Must honour:**
  - One complete document per resource; `nodes[]` and `populated_field_ids[]` from **all** of the
    resource's nodes (a separate all-nodes query), independent of index flags; `field_*` values
    from the fields B1 says.
  - Decide and document the `0` sentinel (the old indexer prepended `0` so the arrays always exist;
    29 no-metadata resources once lacked `populated_field_ids`).
  - `title` from `$view_title_field`; `date_field_sort` from `$date_field`; `creation_date`
    from `resource.creation_date`; `modified_date` from `resource.modified`; `access`; `created_by`
    absent when NULL; `ref_s` for number-as-text matching.
  - Dates through A4 (`_q`, `_ts`, `_range_*`); date-range fields need both endpoints to form an
    interval; text through A5; numbers as floats; values as native ints where the schema says int.
  - Category-tree ancestors are stored on the resource at save, so hierarchy needs nothing extra.
  - The same function serves the rebuild and the single-resource sync path — a test guards against a
    future fork (sync plan Phase 3).
- **Blocking questions:** Q1.2, Q1.6, Q1.4
- **Proves it:** golden documents for the fixture resources; the document for a ref from the rebuild
  path equals the one from `build_resource_documents([ref])`.

### B3 — Membership document builder
- **Builds:** `build_membership_documents(collection refs or resource refs)`; the indexing rule
  `collection_indexed(collection)`.
- **Size:** S · **Depends on:** A3
- **Must honour:**
  - Id `collection:resource`; NULL `sortorder` → INT32_MIN, NULL `date_added` → 0 (reproduces
    MySQL's NULL placement both ways; without it ~109k rows never imported) (P18).
  - Not indexed: selection (type 2), upload (type 1 / negative ref), share-upload collections; the
    38 per-user "New uploads" collections once escaped the cursor (P27). Request, research and
    smart collections per Q5.7.
  - The expected-count query uses the same rule as the builder.
- **Blocking questions:** Q5.7
- **Proves it:** builder battery; count equals MySQL under the rule on the fixture and the test system.

### B4 — Grant document builder
- **Builds:** `build_grant_documents(resource refs)`.
- **Size:** S · **Depends on:** A3
- **Must honour:**
  - `user` / `usergroup` is `−1` when unused so an anonymous or no-group searcher (0) cannot match
    (a leak found live); group grants never expire; user grants carry `expires` (0 = never) and are
    checked at query time, so an expiry needs no reindex but an edit does (P1).
  - Batches take whole resources (a derived table with DISTINCT resource) so a resource's grants
    cannot straddle a batch boundary.
- **Blocking questions:** Q1.10
- **Proves it:** battery including "anonymous user cannot match a placeholder grant".

### B5 — Generation rebuild
- **Builds:** `include/rebuild.php` and `scripts/reindex.php` — build the three collections as a new
  generation, verify, swap aliases, delete the old generation, report.
- **Size:** M · **Depends on:** A2, A3, B2, B3, B4
- **Must honour:**
  - Build into empty generation collections `<prefix>_<name>_g<N>`; swap all three aliases after a
    verified count check; delete the previous generation immediately (P7); a failed build keeps the
    old generation serving.
  - Queue rows written during the build are stamped and replayed against the new generation before
    the swap (P9); the build takes a lock; the admin job fails if the lock is held (P11).
  - Measured rates: fresh upsert 770–1,800 docs/s, re-upsert 540/s, partial `update` 650–780/s (as
    costly as a full write), four parallel imports ≈ 2×, 2,000-doc chunks no faster than 500,
    memberships 3,300/s (7,800/s parallel); a full run just over a minute for ~115k + ~152k.
  - Improvements to include: one write per resource, build into an empty collection, 3–4 imports in
    flight (`curl_multi`), encode JSON once, bigger MySQL batches for the attributes pass, check the
    title derived table with `EXPLAIN`.
  - Report: progress and ETA, failure reasons per line, a summary with Typesense vs MySQL counts,
    exit 0/1/2. The indexer host must run the new code (a rebuild once ran old code).
  - `--drop` becomes unnecessary (a generation is always fresh); keep an explicit "rebuild now".
- **Blocking questions:** Q8.1, Q8.10, Q9.3, Q9.4, Q1.11
- **Proves it:** rebuild on the local fixture gives exact counts; rebuild on the test system within
  the agreed budget; a search issued during the swap never returns empty.

### B6 — Schema version and migration
- **Builds:** version recorded with each generation (alias metadata or a meta document); mismatch
  detection → "rebuild required"; an additive `PATCH` path for small changes between rebuilds.
- **Size:** S · **Depends on:** B5
- **Must honour:**
  - One `PATCH` can add and drop fields on a live collection and can make a field optional by
    drop-and-re-add while keeping values (verified); use it only for additive changes.
  - A field-config change, a `$stemming` change or a schema-version bump triggers a full rebuild.
- **Blocking questions:** Q1.8
- **Proves it:** an upgrade test from a previous schema version; a version mismatch is detected and
  reported on the setup page.
