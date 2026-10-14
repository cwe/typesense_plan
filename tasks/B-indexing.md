# B — Indexing

Milestone M1: an index built from scratch whose counts equal MySQL. Nine tasks. The map of what the
index holds — every field and representation, its source, its freshness event and the package that
needs it — is [B-schema-register.md](B-schema-register.md); it plays the role the catalogue plays
for the search packages.

## The contract: the index is extended by registration, not modification

Later packages add fields to the index (the date components for the dropdowns, the option-name
array, the string-comparison date key, the `!properties`, `!related`, image, flag, geo and colour
fields, the sort keys, type names). None of them reopens the builder, the rebuild or the worker:

1. **B2 is the extension point.** Every index field is an entry in the representation registry:
   its name or pattern, Typesense type and flags, the field types and index flags it applies to,
   the value function, its source table, the schema version it arrived in, and the package that
   needs it. The builder (B4) walks the registry; it knows nothing about any particular field.
2. **The schema-change protocol** for a package that needs a field: add the registry entry and
   its value function; add the row to the schema register; bump `schema_version()`; add the
   value-function case to B2's battery and a golden-document case to B4's; run the builder-path
   equality test; rebuild on the fixture. Deployment is a generation rebuild (G5, B9); until it
   runs, B9 reports "rebuild required" and the nightly build picks the new schema up.
3. **The freshness rule.** A representation whose source is `resource`, `resource_node` or
   `resource_data` is kept fresh by the events already in the event register. One whose source is
   another table (`resource_dimensions`, `resource_related`, `daily_stat`, `resource.lock_user`
   set by a path without a hook) must add its event to [C-event-register.md](C-event-register.md)
   with a hook or an interim ("nightly rebuild") before the package closes.
4. **Batching.** A package may land its entry alone (a rebuild is ~1 minute on the test system) or
   several may share one schema version (Q4.2). Either way the register says which version carried
   what.

### B1 — Field inventory
- **Covers:** register section 4 (the field-type matrix), the "applies to" column of section 2.
- **Builds:** `include/fields.php` — every `resource_type_field` with ref, short name, type,
  `keywords_index`, `partial_index`, `complete_index`, `active`, `field_constraint`, resource types,
  the view-permission check; cached per request; the lookup **returns** every flag.
- **Size:** S · **Depends on:** A1 · **Milestone:** M1
- **Must honour:** an earlier fix was inert because the lookup selected a flag but did not return
  it — test through the lookup; inactive fields are listed (the register needs them for Q6.7) but
  marked, and excluded from positive matching (P37); `field_constraint == 1` is the numeric flag
  the search UI and core's `+0` sort share; the short-name match rules are Q3.17; `basic*` names
  are reserved (Q3.13).
- **Fixture:** the base fixture's 15 fields cover every type and flag the matrix lists except a
  complete-indexed field and a non-indexed fixed-list field (add both).
- **Battery:** `harness/ab/B1_field_inventory.php` — one case per matrix row: the lookup returns
  the expected flags, visibility and numeric flag; a hidden field (`f-93`) and an inactive one.
- **Blocking questions:** Q1.1, Q3.11, Q3.17

### B2 — Representation registry and schema version
- **Covers:** register sections 1–3 (every field of the three collections) and section 5.
- **Builds:** `include/representations.php` — the registry (resource-level fields, per-field
  patterns `field_<ref>_<suffix>`, membership and grant fields): for each entry the name or
  pattern, type, `optional` / `facet` / `sort` / `stem` flags, the (field type, index flag)
  selector, the value function, the source, the version it arrived in, the owning package;
  `schema_version()` derived from the registry; `fields_for(field)` used by B4; the schema
  fragments A3 assembles into the three collection schemas; `scripts/schema.php --register`
  prints the register table so the document cannot drift from the code.
- **Size:** M · **Depends on:** A3, A4, A5, B1 · **Milestone:** M1
- **Must honour:** dynamic patterns `field_.*_<suffix>`, explicit fields where a per-field option
  is needed; Typesense rejects a numeric string for an int field; `_q`, `_ss` option names (if Q3.12
  wants them word-matched they are text, not strings), sort keys and `_k` never stem; text fields
  stem iff `$stemming` — or not at all if Q2.14 pre-stems with `GetStem()`; three sort fields per
  query at most; `token_separators` = `$config_separators` + `,` `:`, single-byte only; the
  numeric `_f` only when `is_numeric` (the `$value == 1` bug); dates through A4 give `_q`, `_ts`
  (full dates), `_range_start/_end` (the value at its own precision; a date-range field's span)
  and, per Q3.4, `_k`; the `date_field_sort` key `0:` NULL, `1:` empty, `2:<value>` (P19).
- **Fixture:** none (unit cases on values).
- **Battery:** `harness/ab/B2_representations.php` — for every registry entry, value in → stored
  value out (the register's rows are the case list); the schema snapshot equals the generated
  register; the local Typesense accepts one sample document per pattern; `schema_version()`
  changes when an entry is added.
- **Blocking questions:** Q1.2, Q1.5, Q2.14, Q3.4, Q3.12
- **Open questions:** Q1.14 (confirm the registry as the extension contract and the protocol)

### B3 — Resource-level fields
- **Covers:** register section 1.
- **Builds:** in `include/documents.php`: the always-present part of a resource document — `ref`,
  `ref_s`, `resource_type`, `archive`, `access`, `created_by` (absent when NULL), `creation_date`,
  `modified_date`, `title` (from `$view_title_field`), `date_field_sort` (from `$date_field`),
  `nodes[]` and `populated_field_ids[]` from the **all-nodes query** (every `resource_node` row of
  the batch, `node.resource_type_field`), independent of index flags.
- **Size:** S · **Depends on:** B2 · **Milestone:** M1
- **Must honour:** `nodes[]` from all nodes is what makes node filters on non-indexed fields and
  `!hasdata` work; category-tree ancestors are stored on the resource at save, so hierarchy needs
  nothing extra; the old indexer prepended a `0` so the arrays always exist (29 no-metadata
  resources once lacked `populated_field_ids`) — Q1.15 decides sentinel or optional field;
  `created_date` was replaced by `creation_date` (day limit) and `date_field_sort` (P19, P20);
  `resource.modified` moves on every `resource_log()` call, so `modified_date` is a coarse signal.
- **Fixture:** the base fixture (a resource with no metadata, one with `created_by` NULL, archive
  and access values of every kind).
- **Battery:** `harness/ab/B3_resource_fields.php` — the skeleton document of every fixture
  resource equals the expected one; a resource with no nodes; the date sort key for NULL, empty,
  year-only and full values.
- **Blocking questions:** Q1.15 (new), Q7.1 (whether `hit_count` / `user_rating` are here)

### B4 — Resource document builder
- **Covers:** register sections 1–2 assembled; the field-type matrix end to end.
- **Builds:** `build_resource_documents(refs[])` — batches of refs; the resource rows (B3), the
  attributes query restricted to the fields B1 says contribute, values grouped per field, each
  field's representations from B2 (text through A5: cleaning, cap, translations per Q1.4; dates
  through A4; numbers as floats); one complete document per resource; the refs missing from MySQL
  returned separately (the delete signal for C2). **The same function serves the rebuild and the
  sync path.**
- **Size:** M · **Depends on:** B3, A4, A5 · **Milestone:** M1
- **Must honour:** one write per resource (the old indexer wrote twice); bigger MySQL batches for
  the attributes pass (the rebuild's measured bottleneck); a date-range field needs both endpoints
  to form an interval; a text value with a BOM is cleaned (P33); the first
  `$node_keyword_index_chars` characters per Q1.13; raw HTML and `~en:~fr:` values per Q1.4; a
  value `abc` in a numeric field goes to `_s` only; int fields as ints; the document must not
  depend on who runs the builder (no user permissions in it).
- **Fixture:** the base fixture, which has every field type the matrix lists except the two B1
  adds.
- **Battery:** `harness/ab/B4_resource_builder.php` — golden documents for every fixture resource
  (every matrix row appears in at least one); the no-metadata resource; a resource after a value
  is removed; a batch boundary falling inside a resource's values; later, the builder-path
  equality case against B8.
- **Blocking questions:** Q1.4, Q1.13

### B5 — Membership documents
- **Covers:** register section 3 (memberships).
- **Builds:** `build_membership_documents(collection refs | resource refs)` and the indexing rule
  `collection_indexed(collection)`; the expected-count query uses the same rule.
- **Size:** S · **Depends on:** B2 · **Milestone:** M1
- **Must honour:** id `collection:resource` (de-duplicates `collection_resource`, which has no
  unique key); NULL `sortorder` → INT32_MIN, NULL `date_added` → 0 (reproduces MySQL's NULL
  placement both ways; ~109k rows never imported without it, P18); `collection_type` carried on
  the document; not indexed: selection (type 2), upload (type 1 / negative ref), share-upload
  collections (P27) — the 38 per-user "New uploads" collections once escaped the cursor; request,
  research and smart collections per Q5.7; keyset pagination by (collection, resource).
- **Fixture:** the base fixture's collections plus one of each excluded type.
- **Battery:** `harness/ab/B5_memberships.php` — documents for each collection kind; the count
  equals the rule's MySQL count on the fixture; live, on the test system.
- **Blocking questions:** Q5.7

### B6 — Grant documents
- **Covers:** register section 3 (grants).
- **Builds:** `build_grant_documents(resource refs)`.
- **Size:** S · **Depends on:** B2 · **Milestone:** M1
- **Must honour:** `user` / `usergroup` is `−1` when unused so an anonymous or no-group searcher
  (0) cannot match a placeholder (a leak found live); group grants never expire; user grants carry
  `expires` (0 = never), checked at query time, so an expiry needs no reindex but an edit does
  (P1); batches take whole resources (a derived table with DISTINCT resource) so a resource's
  grants never straddle a batch; `copy_resource` writes junk `(NULL, NULL)` rows — skip them;
  `copy_locked_data` duplicates rows and drops expiry.
- **Fixture:** the base fixture's grants (a group grant, an expired user grant) plus a junk row.
- **Battery:** `harness/ab/B6_grants.php` — the placeholder rule, expiry, the junk row skipped,
  the whole-resource batch, counts.
- **Blocking questions:** Q1.10

### B7 — Import and count verification
- **Covers:** the transport every builder's output takes.
- **Builds:** `include/import.php` — `import_documents(collection, docs, action)` in chunks
  (`$typesense_search_import_max_docs` / `_max_bytes`), JSON encoded once, N imports in flight via
  `curl_multi` (`$typesense_search_import_concurrency`), per-line result parsing with each
  failure reported, 503 back-off with halved concurrency (P16), a timing report (docs/s per
  collection); `verify_generation(N)` — Typesense `num_documents` against the builders' expected
  MySQL counts within a tolerance, plus one sample reference join through the real names.
- **Size:** M · **Depends on:** A2 · **Milestone:** M1
- **Must honour:** `update` on a missing id fails per line inside an HTTP 200 (resource failures
  cascade into membership and grant reference failures); `per_page`/chunk caps; concurrency never
  above the server's cores − 2; leave the server's own batch size alone; fresh upsert 770–1,800
  docs/s, re-upsert 540/s, four parallel ≈ 2×, 2,000-doc chunks no faster than 500 (P16); never
  create a collection here.
- **Fixture:** none (documents from B4–B6 on the fixture).
- **Battery:** `harness/ab/B7_import_verify.php` — a chunk with one bad line reports that line; a
  422 and a simulated 503 are handled; concurrency halves; the count check passes on a complete
  generation and fails on a short one; the timing report has every collection.
- **Blocking questions:** Q8.6, Q9.4, Q9.10 (concurrency only if the budget needs it)

### B8 — Generation rebuild and the reindex command
- **Covers:** the rebuild steps of the sync plan §6.8 and the admin job of §6.9.
- **Builds:** `include/rebuild.php`, `scripts/reindex.php`, the offline job handler
  (`job_handlers/typesense_search_reindex.php`) that runs the identical build: take the build lock;
  record `building_generation`; create the three `_g<N+1>` collections from B2's schema with
  metadata `{generation, schema_version, plugin_version, instance (A8), started}`, `async_reference`,
  `cascade_delete` — and the clash check before creating anything; stream
  resources, memberships, grants (and synonyms, if SA10 chooses them) through B4–B7; **catch-up**
  of queue rows stamped during the build (a seam that calls C2's `apply` — a no-op until C2
  exists); pause the worker, final catch-up, verify, mark complete, swap all three aliases, set
  `live_generation`, clear `building_generation`, set `ready`, resume the worker; delete the old
  generation; on failure no swap, the new collections deleted, an admin notified; the one-off
  migration from un-aliased names; progress, ETA, a summary with counts per collection, exit codes
  0/1/2; `--verify-only`.
- **Size:** M · **Depends on:** B4, B5, B6, B7, A3, A6 · **Milestone:** M1
- **Must honour:** a reference binds to the real collection at creation, so memberships and grants
  are created against the generation's resources collection and all three aliases move together
  (moving one breaks joins); the lock refuses a second build and the admin job fails with a message
  rather than rescheduling (P11); a failed build keeps the old generation serving; the old
  generation is deleted immediately after a verified swap (P7); the indexer host must run the new
  code (a rebuild once ran old code: record the plugin version in the metadata and refuse a
  mismatch); `--drop` is unnecessary (a generation is always fresh); a search issued during the
  swap must never return empty.
- **Fixture:** the base fixture; the harness can interleave a search during the swap and simulate
  a verification failure.
- **Battery:** `harness/ab/B8_rebuild.php` — exact counts on the fixture; generation number
  advances and the old one is gone; a search during the swap is served or falls back, never empty;
  a failed verification keeps the old generation; the lock; the builder-path equality case (the
  document for a ref from the rebuild equals `build_resource_documents([ref])`); live: the test
  system rebuilt within the agreed budget with its timing report.
- **Blocking questions:** Q8.1, Q8.10, Q9.3, Q9.4, Q1.11, Q0.7

### B9 — Schema version and migration
- **Covers:** register section 5 (versions).
- **Builds:** the version in each generation's metadata; at boot and before serving, compare it
  with `schema_version()` → a "rebuild required" state the setup page shows (G1) and the nightly
  build acts on; `scripts/reindex.php --patch` for additive changes (add or drop a field on the
  live collection, make a field optional by drop-and-re-add keeping values — verified) with the
  rule that only additive changes use it; the triggers that force a rebuild (`$stemming`, a
  field-config change moving a field between representations — detected by C7's snapshot diff —
  and a version bump).
- **Size:** S · **Depends on:** B8, A6 · **Milestone:** M1
- **Must honour:** a PATCH needed a 600 s timeout on the test index; the version mismatch must not
  make searches fail — serve the old generation until the rebuild, unless the mismatch is one the
  query code cannot work with (then `ready = false`).
- **Battery:** `harness/ab/B9_schema_version.php` — an upgrade from a previous version on the
  fixture; a mismatch is detected and reported; a PATCH adds a field and keeps values; a search
  served during the mismatch.
- **Blocking questions:** Q1.8
