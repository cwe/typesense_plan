# I — Core changes

I1 is milestone M0 (the plugin cannot work without it); I2 is M5; I3–I5 are M7. Every item here
depends on the policy in Q0.4 / Q11.1.

### I1 — The hook-return patch
- **Builds:** `include/do_search.php`: the `external_search` hook's return handling, as a documented
  fork patch (`CORE_PATCHES.md`) or an upstream PR.
- **Size:** S · **Depends on:** —
- **Must honour:** when the provider returns anything other than `false` — including an empty set —
  core returns it; `false` lets core continue to `search_special()` and the standard query; the hook
  fires before both, with `$sql_filter`, `$sql_join` and `$select` already built (the keyword
  criteria are baked into them).
- **Blocking questions:** Q11.1, Q11.2
- **Proves it:** a stock RS checkout plus this patch runs the plugin; `CORE_PATCHES.md` lists it with
  its reason.

### I2 — The collection access gate helper (G1)
- **Builds:** a shared function in `include/search_functions.php` extracted from
  `search_special()`'s `!collection` branch (own, public, featured with access control, request,
  research, upload-share session collections, share key, `$ignore_collection_access`,
  `collection_readable()`), used by core and the plugin.
- **Size:** S · **Depends on:** I1
- **Must honour:** no behaviour change in core; the plugin's copy (G2) is the fallback if Q5.1 says
  no core change, accepting drift.
- **Blocking questions:** Q5.1, Q11.1
- **Proves it:** core's collection battery unchanged; the plugin's gate battery passes.

### I3 — Core hooks for sync (Phase 2)
- **Builds:** one PR: node-layer `after_resource_nodes_changed`, `after_set_node`, before/after node
  delete; `after_update_resource_type`; `after_resource_custom_access_change`;
  `after_put_resource_data`; collection bulk hooks after the writes in `update_collection_order`,
  `delete_collection`, `remove_all_resources_from_collection`, `add_saved_search_items`,
  `copy_collection`, the type changers; arguments on the field-config hooks.
- **Size:** M · **Depends on:** I1
- **Must honour:** the node layer has zero hooks today; the collection add/remove hooks fire before
  the write; custom-access writers via requests and collection email have no hook and no log;
  `resource.modified` moves only through `resource_log()`.
- **Blocking questions:** Q8.4, Q11.1
- **Proves it:** C6's handlers make the hook-less freshness cases pass.

### I4 — Core bug dispositions
- **Builds:** a table in this repo and in `CORE_PATCHES.md`: for each item — raise upstream, copy
  the behaviour, fix in the fork, or ignore — with the reason.
- **Size:** S · **Depends on:** —
- **The list:** `!last` + node bucket returns one row (Core 1); date-range-field exact search has
  reversed SQL parameters (Core 2); `!resource` takes every digit (Core 3); `!empty<name>` binds the
  name as an integer, always field 1 (Core 5); `!empty` of a non-global field puts the type
  condition inside the NOT IN (Core 6); `!lastN term` with a space → 1000 (Core 7); ordering by a
  non-join metadata field returns nothing (Core 8); bare `a;b` OR works only after a field search
  created the combined keyword (Core 9); a no-whitespace comma string is a phrase (Core 10); an
  unknown or misspelt special returns everything; `csv_export_results_metadata.php` passes `null`
  to a typed hook parameter; `!related` over 120 s; the form's `!properties` + `!list` collision;
  `startdate:`/`enddate:` dead code; `$ignore_filters` effectively dead; `view.php` next/previous
  fetching all rows; the page offset with collections above results; the `J` join's missing
  featured-type condition.
- **Blocking questions:** Q11.3, Q7.3, Q7.5, Q4.8, Q4.10
- **Proves it:** every item has a row; the chosen PRs or issues are filed and referenced.

### I5 — Tracking upstream
- **Builds:** the procedure for following ResourceSpace releases (rebase or merge `typesense`,
  re-check the patch list, re-run H7), and who does it.
- **Size:** S · **Depends on:** I1
- **Blocking questions:** Q11.4, Q11.6
- **Proves it:** one release followed by the procedure.
