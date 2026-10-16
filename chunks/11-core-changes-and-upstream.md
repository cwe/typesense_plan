# Chunk 11 — Core changes and upstream

Status: planning. Depends on: 0 (Q0.4). Feeds: 5, 6, 7, 8.

## Scope

Every change the plugin needs or wants in ResourceSpace core itself, the core bugs found along the
way, and the policy for each: fork-only, upstream, replicate, or ignore.

## What we know

- The fork already carries a core patch: `include/do_search.php` enables the `external_search` hook's
  return handling (it was commented out — "just from my testing"). Without it the plugin's results are
  ignored, so stock RS cannot use the plugin.
- Core changes the plans want: a shared `!collection` access gate (G1); hooks — node layer
  (`after_resource_nodes_changed`, `after_set_node`, before/after node delete),
  `after_update_resource_type`, `after_resource_custom_access_change`, `after_put_resource_data`,
  collection bulk writes (reorder, delete, empty, add-all, type change, copy), arguments on the
  field-config hooks; a windowed next/previous query (Q7.3); an aligned page offset when
  collections appear above results (Q7.5); the `J` join's missing featured-type condition (`j*`
  quirk).
- Core bugs found (review Core 1–10 and later): `!last` + a node bucket returns one row (aggregate
  without GROUP BY); a date-range-field exact search has its SQL parameters reversed; `!resource`
  takes digits from the whole string; `!empty<name>` binds the name as an integer (always field 1)
  and puts a non-global field's type condition inside the NOT IN; `!lastN term` with a space → 1000;
  ordering by a non-join metadata field returns nothing; bare `a;b` OR works only after a field
  search created the combined keyword (a search changes the index); a no-whitespace comma string is
  a phrase; an unknown or misspelt special returns everything; `csv_export_results_metadata.php`
  passes `null` for a typed hook parameter; `!related` can exceed 120 s; the advanced form's
  `!properties` + `!list` collision; `startdate:`/`enddate:` dead code; `$ignore_filters` is
  effectively dead.
- Chris's decision so far (2 Oct): core bugs left as they are, review only.
- Upstream is Montala's ResourceSpace; the plugin's yaml names Montala as author and marks it
  experimental, so it may be intended to go upstream, where core changes would not be available.

## Candidate work items

- A `CORE_PATCHES.md` in the plugin (or here) listing every fork change with its reason and upstream
  status.
- One core PR for the hooks (sync plan Phase 2) and one for the helper, if Q0.4 allows.
- Issues or PRs upstream for the bugs chosen in Q11.3.

## Questions to answer before coding

### Q11.1 — The policy, by category
For each: fork-only, upstream PR, or neither. Categories: the hook-return patch; new hooks; the
collection gate helper; the next/previous and page-offset fixes; bug fixes; behaviour fixes to
quirks. (Q0.4 asks this in general; answer it here per category.)

### Q11.2 — The existing hook-return patch
Is it a prerequisite that must go upstream for the plugin to work on stock RS? Was it commented out
upstream for a reason worth knowing before proposing it?

### Q11.3 — Each core bug: raise, copy, or ignore
Go through the list above and mark each. The ones that change what users see most: unknown special
→ everything; `!empty` by name; the bare-OR side effect; `!last` + nodes; the CSV export TypeError
(which can also be fixed on the plugin side by making the parameter nullable).
**Catalogue marks (2026-10-16):** two core-bug rows are *must* — F5 (`!last` with a node bucket
returns one row, Core 1) and F47 (`!empty` of a non-global field, Core 6): *must* on a bug means
replicate it or fix it, which this question decides. F31 (Core 3) is *check*; F46 and F49 (Core 5)
are not required.

### Q11.4 — Tracking upstream
How the fork follows RS releases (rebase `typesense` onto each release? merge?), who does it, and
how the patch list is checked after each one.

### Q11.5 — A cleaner provider contract
The hook hands the plugin `$sql_filter` with keyword criteria already mixed into the permission
filter, which is why read-time validation must rebuild the permission part. Propose upstream a
provider hook that receives the permission filter separately (and structured criteria)? Or keep the
plugin working with the hook as it is?

### Q11.6 — Is this plugin meant to go upstream?
If yes, the constraints are: no core changes beyond what upstream accepts, no fork-only behaviour.
That decision changes the answers to most of this chunk.

## Done when

A policy per category is recorded; the patch list exists and is complete; each core bug has a
disposition; any PRs or issues are filed and referenced.
