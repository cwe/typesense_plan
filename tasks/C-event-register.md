# Event register

Every write in core that changes what a search returns, with the hook a plugin can use today, the
queue row it becomes, the task that handles it, the interim when there is no hook, and the freshness
case that proves it. The map for area C, as [search/coverage.md](search/coverage.md) is for the
search packages. Condensed from the sync plan's §3 (`typesense-incremental-sync-plan.md` on
`typesense`), with the core functions re-checked on 14 October 2026 (`node_functions.php` fires no
hook at all; `save_resource_custom_access()`, `update_resource_type()`, `update_related_resource()`,
`update_collection_order()`, `delete_collection()`, `copy_collection()`, `add_saved_search_items()`,
`remove_all_resources_from_collection()`, `save_collection()`, `update_collection_type()` and
`update_resource_lock()` fire none).

Queue rows are `kind ref (priority)`; priorities: 0 visibility, 1 edit, 2 repair, 3 bulk. **Interim**
is what keeps the index right until the row's hook exists: *rebuild* = the nightly generation
rebuild; *C8* = read-time validation hides a stale row meanwhile (access and state changes only).
*Status* stays blank until the freshness case is green.

## Resource lifecycle (C5)

| # | Event | Core path | Hook today | Queue row | Task | Interim | Freshness case | Status |
|---|---|---|---|---|---|---|---|---|
| E1 | Create | `create_resource()` | `resourcecreate($ref, $type)` — before metadata is written | `resource R (1)` | C5 | — | create via API, search for its title after the later saves |  |
| E2 | Copy, and the normal upload from the user template | `copy_resource()` | `afternewresource($to)` at the end; `resourcecreate` does not fire | `resource R (1)`, `grants R (0)` (copies carry grants, incl. junk rows) | C5 | — | copy a resource with a group grant |  |
| E3 | Upload sequence | `upload_batch.php`, `upload_processing` job | `resourcecreate` / `afternewresource`, `after_update_archive_status` (always), `uploadfilesuccess($resource_ref)`, `afterpluploadfile($ref, $ext)`, `afterpreviewcreation` (maybe from a job), many `update_field` (exif) | one `resource R (1)` after coalescing | C5 | — | an upload fires ~10 hooks and yields one row; searchable after the flush |  |
| E4 | Soft delete (to `$resource_deletion_state`) | `delete_resource()` → `update_archive_status()` | `after_update_archive_status($resources[], $archive, $existing[])` | `resource R (0)`; `memberships_of_resource R (0)` when `$remove_deleted_resources_from_collections` | C5 | C8 | delete; the resource leaves every search and its collections |  |
| E5 | Hard delete (purge, or no deletion state) | `delete_resource()` | `delete_resource_extra($ref)`, `beforedeleteresourcefromdb($ref)`, `afterdeleteresource()` (no args) | `resource R (0)` → C2 resolves as a delete; cascade removes memberships and grants | C5 | C8 | purge; document, memberships and grants gone |  |
| E6 | Archive / workflow change | `update_archive_status()`; `save_resource_data` :1346; batch edit; API | `after_update_archive_status` (fires even if unchanged; logged before the UPDATE) | `resource R… (0)` | C5 | C8 | move to state 2 for a `z2` user; the row vanishes |  |
| E6a | Archive change without a hook | `put_resource_data()` (API), `update_resource()` (`after_update_resource($resourceId)` fires later), staticsync, `remove_missing_files.php`, workflow plugins' raw SQL | none / late | `resource R (0)` from C9's `after_put_resource_data`; `after_update_resource` covers `update_resource()` | C5 (`after_update_resource`), C9 | rebuild + C8 | `put_resource_data` archive=2 → hidden by C8 until the rebuild |  |
| E7 | Access level change | `save_resource_data` :1382; batch :2304; `copy_locked_data`; csv_upload; action_dates | `aftersaveresourcedata` (`$ref` int from single save, `$list` array from batch) | `resource R… (0)`, `grants R… (0)` | C5 | C8 | set access 2; hidden from an ungranted user next request |  |
| E7a | Access change without a hook | `put_resource_data`, `copy_locked_data` (edit.php only), csv_upload, staticsync, action_dates | none | `resource R (0)` from C9 | C9 | rebuild + C8 | |  |
| E8 | Resource type change | `update_resource_type()` (edit form → `aftersaveresourcedata` unless an early return; batch; API; `copy_locked_data`; csv; emu) | none in the function; the edit form's after-hook usually follows | `resource R (0)` from C9's `after_update_resource_type`; meanwhile the edit path's `aftersaveresourcedata` | C5 (edit path), C9 | rebuild | API `update_resource_type` → stale until the rebuild; edit page → fresh |  |

## Metadata writes (C5; gaps C9)

| # | Event | Core path | Hook today | Queue row | Task | Interim | Freshness case | Status |
|---|---|---|---|---|---|---|---|---|
| E9 | Edit page save | `save_resource_data()` | `befsaveresourcedata($ref)`; `aftersaveresourcedata($ref, $nodes_to_add, $nodes_to_remove, …)`; early returns at :682 (locked) and :1239 (field errors) skip it after some writes | `resource R (1)`, `grants R (0)` | C5 | rebuild for the early-return paths | edit the title; search in the next request |  |
| E10 | Batch edit | `save_resource_data_multi()` | `saveextraresourcedata($list)`, `aftersaveresourcedata($list, …)`; early return at :2356 skips both after the writes | `resource R… (1)`, `grants R… (0)` | C5 | rebuild for the early return | batch-edit a field on five resources |  |
| E11 | `update_field()` | API `update_field`, defaults, exif/text extraction, geolocation, text autocomplete, locked fields, csv text columns, most plugins | `update_field($resource, $field, $value, $existing, $fieldinfo, $newnodes, $newvalues)` — not on validation failure or unchanged text | `resource R (1)` | C5 | — | `update_field` a caption word; search |  |
| E12 | Related keywords | `save_related_keywords()` | `after_save_related_keywords($keyword, $related)` | synonym set update (direct) | C7 | — | add a pair; both words match (if SA10 serves synonyms) |  |
| E13 | Preview creation (`has_image`, `colour_key`) | `create_previews()` | `afterpreviewcreation` | `resource R (1)` | C5 | — | needed once SF12 exists; `colour_key` only if SF15 is revived |  |
| E14 | File replace (extension, size, dimensions) | `replace_resource_file()`; `pages/view.php` | `replace_resource_file_extra`, `additional_replace_existing`, `afterreplacefile` (view page, no args) | `resource R (1)` | C5 | rebuild where the hook has no ref | needed once SF9 exists |  |
| E15 | Alternative file upload | `upload_batch.php` | `after_alt_upload($alternative, …)` | none needed (alternatives are not indexed) | — | — | — |  |
| E16 | Lock / unlock | `update_resource_lock()` | none | `resource R (1)` from C9's `after_resource_lock_change` | C9 | rebuild | needed once SF13 exists |  |
| E17 | Integrity check | cron | none | — | — | rebuild | `integrity_fail` is fresh nightly (SF13) |  |
| E18 | Relate / unrelate resources | `update_related_resource()`, `relate_all_resources()` | none | `resource R… (1)` from C9's `after_related_resources_changed` | C9 | rebuild | needed once SF11 exists |  |
| E19 | Direct node writers | API `add_resource_nodes` / `_multi`; csv_upload fixed-list columns; metadata templates / "save all remaining" (`copy_resource_nodes`, `copy_locked_data`); fixed-list autocomplete; annotations; google_vision, clip, faces plugins (inside `afterpreviewcreation`); tools (`nodes_remove_duplicates`, `migrate_fixed_to_text`, `cleanup_invalid_nodes`, `database_prune`, `renumber_resources`, `join_fields`, `remove_html`, staticsync) | none (`node_functions.php` has no hooks) | `resource R… (1)` from C9's `after_resource_nodes_changed` | C9 | rebuild | API `add_resource_nodes` → stale until the rebuild; fresh after C9 |  |

## Node definitions, field and global configuration (C7; gaps C9)

| # | Event | Core path | Hook today | Queue row | Task | Interim | Freshness case | Status |
|---|---|---|---|---|---|---|---|---|
| E20 | Node rename / parent / order | `set_node()` (MFO admin, API, migrations) | none | `node N (3)` from C9's `after_set_node`; the setup page can queue it now | C7, C9 | rebuild (the name sits in every resource's `_ss`) | rename an option used by many resources; MySQL serves until done |  |
| E21 | Node delete | `delete_node()` → `delete_node_resources()` | none | `node N (3)` from C9's `beforenodedelete` (refs read before the delete) | C7, C9 | rebuild | delete an option |  |
| E22 | Node active toggle | `update_node_active_state()` | none | — | — | — | core search ignores `node.active`: nothing to do |  |
| E23 | Field config saved (flags, type, constraint, active, name) | `save_resource_type_field()` | `afterresourcetypefieldeditsave()` — no args; `$GLOBALS['ref']` | `field F (3)` when the snapshot diff shows an index-relevant change | C7 | — (the diff makes the hook usable) | flip `keywords_index`; every resource with values is streamed |  |
| E23a | Field deleted | `delete_resource_type_field()` | `after_delete_resource_type_field()` — no args; rows already gone | `field_deleted F (3)`: refs from Typesense `populated_field_ids:=F` | C7 | — | delete a field; stale representations removed |  |
| E23b | Field created / copied | `create_resource_type_field()`, `admin_copy_field.php` | none | nothing (dynamic patterns) | — | — | — |  |
| E24 | Resource type create / rename / delete | `admin_resource_type_edit.php` | none; delete moves resources via `update_resource_type` (E8) | `full (3)` if type names are indexed (SA9) | C7 | rebuild | rename a type (if `words` exists) |  |
| E25 | `$stemming`, `$partial_index_min_word_length`, `$resource_field_verbatim_keyword_regex`, `$config_separators` | `config.php` edits only; `set_config_option()` has no hook | none | "rebuild required" from the global-config snapshot diff (B9) | C7 | rebuild | change `$stemming`; the setup page warns; the nightly build rebuilds |  |
| E26 | `$view_title_field`, `$date_field` | as E25 | none | `full (3)` from the snapshot diff | C7 | rebuild | change `$date_field`; sort keys and components follow |  |
| E27 | Hit counts, ratings | core cron (`hit_count`, at most once a day); ratings page (`modifyuserrating` hook exists) | `after_copy_hitcount_to_live` (I3, Q7.1 decided 2026-10-20); `modifyuserrating` | `resource R…` (1) | C5, C9 | rebuild until the hook ships | `hit_count` exact after each daily copy; `user_rating` at once |  |

## Collections (C6; gaps C9)

| # | Event | Core path | Hook today | Queue row | Task | Interim | Freshness case | Status |
|---|---|---|---|---|---|---|---|---|
| E30 | Add to a collection | `add_resource_to_collection()` (API, pages, `copy_collection`, smart collections, upload, requests, csv) | `Addtocollectionsuccess(resourceId, collectionId)` — **before** the delete + insert; named arguments | `membership (C, R) (1)`, applied after the write | C6 | — | add; `!collection` shows it; `date_added` and `sortorder` as core |  |
| E31 | Remove from a collection | `remove_resource_from_collection()` | `Removefromcollectionsuccess` — before the DELETE; fires even when not a member | `membership (C, R) (1)` → delete | C6 | — | remove; gone from `!collection` |  |
| E32 | Reorder | `update_collection_order()` (NULL `sortorder` → 99999 for the rest) | none | `collection C (3)` from C9 | C9 | rebuild | reorder; collection order stale until the rebuild (Q5.9) |  |
| E33 | "Add all results" | `add_saved_search_items()` (shifts every `sortorder`, re-adds) | none | `collection C (3)` from C9 | C9 | rebuild | |  |
| E34 | Empty a collection | `remove_all_resources_from_collection()`; selection cleared on a new search or logout; upload review reset | none | `collection C (3)` from C9; selection collections need nothing (not indexed) | C9 | rebuild | |  |
| E35 | Copy with wipe; upload-review clean-up | `copy_collection(…, true)`; `collection_cleanup_inaccessible_resources()` | none | `collection C (3)` from C9 | C9 | rebuild | |  |
| E36 | Collection type / parent / public / user change | `save_collection()`, `update_collection_type()`, `collection_set_public()`, `update_collection_user()` | none | `collection C (3)` from C9 (`collection_type` sits on every membership document) | C9 | rebuild | make a collection featured; `j*` users see it after the rebuild |  |
| E37 | Delete a collection | `delete_collection()`; `cleanup_anonymous_collections()` (cron) orphans memberships | none | `collection C (3)` from C9 → delete by filter | C9 | rebuild | |  |
| E38 | Smart collection refresh | `update_smart_collection()` from `search_special('!collection')` | via E30/E31 per resource | as E30/E31 | C6 | — | SF2's smart-collection case (Q4.6) |  |
| E39 | Memberships of a soft-deleted resource | `update_archive_status()` raw delete | `after_update_archive_status` (E4) | `memberships_of_resource R (0)` | C5 | C8 | as E4 |  |
| E39a | Raw SQL writers | staticsync, action_dates, research copy, `database_prune`, `renumber_resources` | none | — | — | rebuild | |  |

## Custom access grants (C6; gaps C9)

| # | Event | Core path | Hook today | Queue row | Task | Interim | Freshness case | Status |
|---|---|---|---|---|---|---|---|---|
| E40 | Edit page / batch custom access | `save_resource_custom_access()` (deletes all group rows, inserts from the form) | `aftersaveresourcedata` afterwards (cannot tell whether grants changed) | `grants R (0)` always | C6 | C8 | grant a group; the group sees the resource next request |  |
| E41 | Leaving access 3 | `delete_resource_custom_access_usergroups()` | `aftersaveresourcedata` | `grants R (0)` | C6 | C8 | set access 0; group grant rows gone |  |
| E42 | Email a resource with access | `open_access_to_group()` via `resolve_open_access()` | `additional_email_resource` (skipped if the send fails) | `grants R (0)` | C6 | rebuild + C8 | |  |
| E43 | Request approval; "grant internal access" on a collection email | `open_access_to_user()` (sets `user_expires`) | none usable (`saverequest` is a pre-hook; `additional_email_collection` has no resource list) | `grants R (0)` from C9's `after_resource_custom_access_change` | C9 | rebuild (the grant is invisible until then — the user cannot see the resource, the safe direction) | approve a request |  |
| E44 | Request declined / back to pending | `remove_access_to_user()` | none | `grants R (0)` from C9 | C9 | rebuild + C8 | decline; C8 hides the resource meanwhile |  |
| E45 | Resource share page delete; `pages/ajax/remove_custom_access.php` | `delete_resource_custom_user_access()`; raw delete | none | `grants R (0)` from C9 | C9 | rebuild + C8 | |  |
| E46 | Copy / locked-field copy | `copy_resource` (junk `(NULL, NULL)` rows), `copy_locked_data` (duplicates, drops expiry) | `afternewresource` / edit page only | `grants R (0)` (E2) | C5 | rebuild for `copy_locked_data` | |  |
| E47 | csv_upload grants | csv_upload :424 | none | `grants R (0)` from C9 | C9 | rebuild | |  |
| E48 | Grant expiry | time passes | — | nothing: `expires` is checked at query time (P1) | D6 | — | an expired user grant stops matching with no sync |  |
| E49 | User or group deleted | `save_user` (`on_delete_user`), `delete_usergroup()` | rows left in place | nothing to do for the index | — | — | — |  |

## Rebuild and outage (B8, C4)

| # | Event | Handling | Task | Freshness case | Status |
|---|---|---|---|---|---|
| E50 | Edits during a build | rows stamped with the generation, replayed against the new collections before the swap, reprocessed by the worker after it (P9) | B8, C2 | edit during a build; correct in the new generation |  |
| E51 | Typesense down | breaker opens; searches fall back; rows accumulate; worker retries with backoff; 503 pauses bulk work (P16) | A2, C4 | stop, edit, restart, drain; never an empty result |  |
| E52 | Build failure | no swap; old generation keeps serving; new collections deleted; admin notified | B8 | simulated verification failure |  |
| E53 | Alias missing / generation mismatch | `ready = false`; searches fall back; admin notified | C4, B9 | delete an alias on the local Typesense |  |
