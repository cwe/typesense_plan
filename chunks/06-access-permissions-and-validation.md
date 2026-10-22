# Chunk 6 — Access, permissions and read-time validation

Status: planning. Depends on: 0, 8. Feeds: 10.

## Scope

Everything that decides whether a user may see a resource in results: access levels and grants,
group search filters, workflow-state permissions, pending rules, resource-type permissions, the
places where core uses a search *as* an access decision, validating index rows against the database
at read time, and the editable-only search.

## What we know

- Replicated and exercised: confidential/custom access via the grants join (user grants expire at
  query time, group grants do not, `−1` placeholder so anonymous users cannot match); `$access` for
  `v` users; group search filter rules (ALL/NONE/ANY, `nodes_on`/`nodes_off`, OR'd with the
  custom-access override and `$open_access_for_contributor`); `T` and `X` type permissions; `z`/`e`
  states and `$additional_archive_states`; `$uploader_view_override`; pending rules with `ert` as
  one OR group; `$k` with `$collection_allow_not_approved_share`; `resource_created_by_filter`; the
  day limit on `creation_date`; `!contributions` open access; `J` featured collections; hidden and
  (staged) inactive fields excluded from positive matching.
- Known holes: the group filter's custom-access override ignores grant expiry; the group filter is
  skipped under `access_override` (core applies it; latent — the only caller passes `smartsearch`,
  which vetoes); `J` with no `j` permissions throws a TypeError (core returns nothing);
  `$ignore_filters` makes the standard restriction switch itself off (core uses the flag only for
  keyword parsing — a one-line fix); `$heightmin` not replicated; NOT over hidden/inactive fields
  differs by a handful.
- **No read-time validation**: hydrate fetches rows by `ref IN (...)` and never re-applies core's
  permission filter. A resource whose state, access, grant, filter node or type changed since the
  last reindex is still listed (existence, title, preview; the file itself is still gated by
  `get_resource_access()`).
- Core uses `do_search()` as an access gate: `get_resource_access()` runs `!resource<ref>` for users
  with a search filter (the plugin's index then decides per-resource access); the collection share
  page requires `!collection` to return every member; requests copy only what `!collection` returns;
  edit gates compare an all-resources search with an editable-only one (always core) — mixed
  engines; submit-pending lists `!contributions` in state −2; the selection clear uses `!collection`
  on the selection (not indexed → core).
- Typesense-only mode fails closed: an outage or declined form → empty → "confidential" for every
  resource for users with a search filter, empty submit-pending, refused shares.
- The hook declares `bool $smartsearch`; `csv_export_results_metadata.php` passes `null` → TypeError
  on every CSV metadata export, for every group, even with the plugin disabled.
- Editable-only: everything in core's editable block is expressible with existing fields except the
  access=1 "no grant exists" branch (needs per-search grant ref lists) and the `modifysearcheditable`
  hook (grant_edit plugin). Used by user-action links, add-editable-to-collection, the actions
  dropdown and refineresults. Earlier recommendation: defer.

## Candidate work items

- Read-time validation in hydrate (Q6.1), with the permission part of `$sql_filter` built
  independently of the keyword part.
- The small fixes: `ignore_filters`, grant expiry in the group filter override, group filter under
  `access_override`, `J` TypeError, the CSV export TypeError (nullable hook parameters, and an audit
  of every hook parameter type against core's call sites).
- Decide and document which gate uses may never be served from the index.

## Questions to answer before coding

### Q6.1 — Validation strictness (open decision D1)
(A) drop stale rows, keep the Typesense total, queue a repair; (B) fall back to MySQL for the whole
search when any row is dropped, queue a repair, count it. (B) was recommended: a dropped row means the
index is stale, so the total and the paging are wrong too. Confirm (B), and confirm the mechanism:
re-run `search_filter()` plus the `rca`/`rca2` joins in the hydrate query (the hook's `$sql_filter`
already contains keyword criteria, so the permission part is rebuilt separately). Cost per search is
one extra join set.

### Q6.2 — Gate uses that must not depend on the index
Candidates to force to core regardless of the plugin: `get_resource_access()`'s `!resource<ref>`
(a per-resource access decision), the share gate, request copies, submit-pending. Options: detect
them (single-resource search, refs-only or count-only `fetchrows`) and veto; or rely on validation
(Q6.1) and accept the residual window; or a core change so these call sites bypass external
providers. Which?

**Decided 2026-10-22 (with Q12.4): a core change.** The gate call sites — `get_resource_access()`'s
`!resource<ref>`, the `!collection<ref>` checks in the request, sharing and edit paths — bypass the
`external_search` hook; the plugin never sees them. `get_resource_access()`'s search is a primary-key
lookup plus the filter conditions, so nothing is lost. Mechanism chosen when I1 is committed.

### Q6.3 — The small fixes
Approve as a batch: `ignore_filters` no longer disables restrictions; grant expiry honoured in the
group filter's override; group filter applied under `access_override`; `J` with no `j` → nothing;
hook parameters made nullable where core passes null.

### Q6.4 — Editable-only searches
Defer until sync and validation land (recommended), or build now (per-search grant ref lists, the
grant_edit hook as a veto or OR-in)? Meanwhile the edit gates compare a plugin count with a core
count — acceptable?
**Catalogue marks (2026-10-16):** G38 is *must*, satisfied by declining (core answers) until this
question builds editable-only searches.

**Catalogue marks (2026-10-20):** G38 is *check* — core's editable-only behaviour is confirmed before the
mark is final; the decline stands meanwhile.

### Q6.5 — Typesense-only mode and access
If Q0.6 keeps it as a testing aid, document "never for real users" and make the setup page say so.
If it becomes a mode, which gate uses must still fall back even in that mode?

### Q6.6 — `$heightmin`
Replicate (needs dimensions indexed, Q1.6) or veto when it is set?

### Q6.7 — NOT over hidden and inactive fields
Core's omit covers all fields; the plugin's covers the indexed, visible ones. Index inactive/hidden
field values for negative matching only (cost, complexity), or accept the small difference?
**Direction from the acceptance rule (Q10.3, 2026-10-16):** a set difference (33 resources on
`-landscape`), so either the words of inactive and hidden fields are indexed for negation only, or
core's reading of inactive fields in a NOT is treated as a defect under Q11.3. Still to say which.

### Q6.8 — Share keys and external access
Collection share keys go through `collection_readable()`; resource-level external keys
(`$internal_share_access`, `k` on a resource) — confirm they are covered or out of scope.

### Q6.9 — Config parity for access options
`$open_access_for_contributor`, `$custom_access_overrides_search_filter`, `$uploader_view_override`,
`$edit_access_for_contributor`, `$collections_omit_archived`, `$search_all_workflow_states`: add a
config matrix to the harness and treat any mismatch as a bug. Agree?

### Q6.10 — Validation and the sync plan
Validation also queues repair rows (sync plan §5.7). Confirm validation ships in sync Phase 0 (not
later), since it is the only protection until the hooks work.

## Done when

The access battery and a config matrix pass in the harness; live A/B with the permission user matrix
(standard, `J`/`j*`, `R`/`h`, `e2`, `ert`, search filter, custom grant) is identical; a stale-index
test (change access, search before the sync runs) shows the fallback, never the resource.
