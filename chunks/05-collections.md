# Chunk 5 — Collections

Status: planning. Depends on: 0, 8 (freshness), 11 (core helper). Feeds: 10.

## Scope

`!collection<ref>` views: the access gate, ordering, the `J` featured-collections restriction inside a
collection, the columns core adds to collection rows, relevance inside a collection, upload-share
and selection collections, and (if in scope) searching *for* collections.

## What we know

- Done (collection parity plan, Phase 0): tie-break order `sortorder, date_added reversed, ref` in
  both directions (verified against core on three collections); NULL `sortorder` → INT32_MIN and
  NULL `date_added` → 0 (memberships rose from 44,801 to 155,141 once NULLs imported); selection,
  upload and share-upload collections not indexed (collection mode vetoes them); `J` inside a
  collection skips the join when the collection is permitted, falls back otherwise, applies under
  `access_override`; `j*` with no `-j` replicates core's any-collection join (104,753 resources);
  pending/archive rules (`ert`, `$uploader_view_override`, `$k` + `$collection_allow_not_approved_share`);
  collection ID parsed as core does; `fetchrows` windows up to `$typesense_search_max_rows`.
- Not done: the shared access gate (G1 core helper vs G2 plugin copy — the plugin's gate is still
  looser than core's valid-collection list and is skipped under `access_override`); hydrating
  `c.date_added`, `c.comment`, `commentset` for collection rows (`commentset` is not read anywhere;
  `date_added` is used by collection functions); relevance inside a collection (plugin: ref desc;
  core: hit counts); the upload-share session gate (core allows only the session's collections; the
  plugin serves any collection once a key is valid); `!collection158` errors in core (unexplained).
- Typesense facts: two `$memberships(...)` clauses ANDed must match the **same** membership
  document, so `!collection<C>` + a `J` join returns 0 unless C is itself permitted (hence the skip);
  joins to different collections work; `id:=[…]` filters work on memberships; the collection sort
  uses all three sort slots.
- Core write sites: only `Addtocollectionsuccess` / `Removefromcollectionsuccess` have hooks, and
  they fire **before** the write with named arguments; reorder, delete, empty, add-all, type change,
  copy and staticsync have none. Freshness is now the sync plan's job (prior decision P17).
- Searching *for* collections (`search_public_collections()`, the collections shown above results)
  is MySQL-only and was out of scope; collections themselves are not indexed. The search page
  passes `[offset − colcount, n]` when collections appear, so page 2+ is misaligned (chunk 7).
- Core quirks: resource types are ignored for collection searches; a type list starting with
  `Global` is ignored entirely.
- A public collection with >250 members and 180+ ties (collection 108) exists for testing.

## Candidate work items

- The access gate (Q5.1), the hydrate join (Q5.2), the upload-share gate (Q5.4).
- Collection cases in the harness (the parity plan's Phase 2 list: user matrix × collection kinds ×
  queries × fetchrows shapes).
- Core write hooks for collections in the core PR (chunk 11), if Q0.4 allows.

## Questions to answer before coding

### Q5.1 — Access gate: core helper (G1) or plugin copy (G2)?
G1 extracts core's `search_special` `!collection` gate (user, public, featured, request, research,
session collections, share key) into a shared function — no behaviour change in core, one source of
truth. G2 copies it and drifts. Depends on Q0.4.

### Q5.2 — Hydrate the collection columns
Add a `LEFT JOIN collection_resource` in the hydrate query for `!collection` results so
`c.date_added`, `c.comment` and `commentset` are present as in core. Needed, or is `date_added` the
only one that matters?

### Q5.3 — Relevance order inside a collection
Fall back to core (exact order), or accept the plugin's order as the main search does (Q7.1)?

### Q5.4 — Upload-share sessions
Replicate core's gate (only the session's collections) — needs the session collection list — or
veto when `upload_share_active()`?

### Q5.5 — Is searching *for* collections in scope now?
If yes: a fourth Typesense collection for collection metadata, with its own sync; the "collections
above results" block and the collection search page would use it. If no, record it as out of scope
(Q0.11).

### Q5.6 — Freshness mechanism
Confirm prior decisions P17 and P8: collection changes reach the index through hooks (where they
exist) and the nightly rebuild, with no per-collection fingerprint. Which collection write sites get
new core hooks (reorder, delete, empty, add-all, type change, copy) — see Q11.3.

### Q5.7 — Which collection kinds are indexed
Confirm: selection, upload, share-upload and negative-ref collections are not. What about request,
research and smart collections?

### Q5.8 — `J` with `j*` and no `-j`
Prior decision P23 replicates core's any-collection join. Confirm, or raise the quirk with core
(Q11.3) and serve the featured-only behaviour once core is fixed?

### Q5.9 — Reordering from the search page
`search.php` reorders via `!collection` + `reorder=true` and `update_collection_order()`; confirm the
plugin is not involved in that path (or that it falls back) and that a reorder shows in the next
view (sync).

### Q5.10 — Large collections and `fetchrows = -1`
Bulk internal callers (collection bar, downloads, batch edit, the selection) fetch all rows. Above
`$typesense_search_max_rows` they fall back to core; below it the plugin walks windows. Confirm this
split is acceptable for collections specifically (a 30,000-member collection takes seconds either way).

### Q5.11 — `$smartsearch`
When set, `search_filter()` keeps the archive states the smart collection saved instead of the
defaults (G40). The old plugin fell back for every smart search; the rule is one condition in D5.
Serve it, or keep the fallback until Q4.6 settles what the plugin does with smart collections?

## Done when

The parity plan's Phase 2 matrix runs in the harness and live; collection views are served with core's
order, gate and columns; freshness cases (add, remove, reorder via SQL + rebuild) pass.
