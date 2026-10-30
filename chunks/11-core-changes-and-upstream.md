# Chunk 11 — Core changes and upstream

Status: planning. Depends on: 0 (Q0.4). Feeds: 5, 6, 7, 8.

## Scope

Every change the plugin needs or wants in ResourceSpace core itself, the core bugs found along the
way, and the policy for each: fork-only, upstream, replicate, or ignore.

## What we know

- The fork carries two core changes (checked against the upstream mirror on 2026-10-20):
  `include/do_search.php` moves the `external_search` hook before `search_special()`, so a provider
  sees every search and `false` falls through to the specials and the standard query — upstream fires
  the hook only for standard searches and has always honoured its result (the earlier note that the
  return handling was commented out was wrong); and `include/resource_functions.php` adds
  `get_visible_indexed_fields()`, a helper the old plugin used, to be moved into the plugin or
  generalised in core when D4 is written.
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
- Chris's decision so far (2 Oct): core bugs left as they are, review only. Superseded on 20 October:
  Chris works for Montala and core changes are made in ResourceSpace itself (Q11.1).
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

**Decided 2026-10-20:** core changes are made in ResourceSpace itself — Chris works for Montala — so the
fork-only / upstream split is gone. Hook reorder, sync hooks, the collection gate function, the
next/previous and page-offset changes: core changes in a release, the plugin stating the minimum
version it needs. Bugs and quirks: fixed in core by default, Q11.3 deciding per row what the plugin
does until the fix ships. Also answers Q0.4.

### Q11.2 — The existing hook-return patch
Is it a prerequisite that must go upstream for the plugin to work on stock RS? Was it commented out
upstream for a reason worth knowing before proposing it?

**Decided 2026-10-20:** the reorder goes into core as it stands in the fork — the hook before
`search_special()`, `false` falling through — with no gating and no second hook. Upstream never had
the return handling commented out; the only change is the order.

### Q11.3 — Each core bug: raise, copy, or ignore
Go through the list above and mark each. The ones that change what users see most: unknown special
→ everything; `!empty` by name; the bare-OR side effect; `!last` + nodes; the CSV export TypeError
(which can also be fixed on the plugin side by making the parameter nullable).
**Catalogue marks (2026-10-16):** two core-bug rows are *must* — F5 (`!last` with a node bucket
returns one row, Core 1) and F47 (`!empty` of a non-global field, Core 6): *must* on a bug means
replicate it or fix it, which this question decides. F31 (Core 3) is *check*; F46 and F49 (Core 5)
are not required.

**Catalogue marks (2026-10-20):** every core-bug row now carries the mark *bug* — F5, D24, F31, F46, F49,
F47, F4, F68, H12, G26, G43, A16, H32, I2, F67, I9 and J2 — with the earlier must / check / no kept in
the row's note; this question decides each one. Not marked: Core 4 (no row), D16 (dead code), G41
(pending Chris's check), F23 (slow, not wrong), the CSV TypeError (no row), I1 and J3 (the parser's
rule), A17 (by design), C7 (test data).

**Decided (2026-10-30):** two more bug rows from the checks — B17 (`title:-sculpture` excludes nothing, a NOT against a junk keyword) and I3 (Core 10's field form) — nineteen in all.

**Direction (2026-10-20, from Q11.1):** the default for a bug row is a fix in core; the row then decides
what the plugin does on versions before the fix (copy the wrong result, or decline so core gives it)
and H7's battery becomes a regression test of the fix.

**Added 2026-10-23 (from the check reading, checks-findings.md):** eleven more defects on I4's list —
the wildcard hidden-field clause that discards a hook's clause, the stale loop variable in the
editable-only filter, `!empty`'s `IN ()`, `!integrityfail` with a word, admin `!noningested` without
a filter, range bounds inherited between keywords, top-level-only option matching, keyword rows
created by searches, dropped OR alternatives, `title:-word` excluding nothing, the 19-argument calls.

### Q11.4 — Tracking upstream
How the fork follows RS releases (rebase `typesense` onto each release? merge?), who does it, and
how the patch list is checked after each one.

**Direction (2026-10-20):** no fork to track — the question becomes release alignment: the minimum
ResourceSpace version each plugin version needs, checked at activation, with `CORE_PATCHES.md` as the
list of core changes behind it (I5).

### Q11.5 — A cleaner provider contract
The hook hands the plugin `$sql_filter` with keyword criteria already mixed into the permission
filter, which is why read-time validation must rebuild the permission part. Propose upstream a
provider hook that receives the permission filter separately (and structured criteria)? Or keep the
plugin working with the hook as it is?

**Direction (2026-10-20):** a design question now, not a proposal — if a provider hook that receives the
permission filter and the parsed search separately is wanted, it is built in core (I6) and C8's
rebuild-the-filter step goes.

### Q11.6 — Is this plugin meant to go upstream?
If yes, the constraints are: no core changes beyond what upstream accepts, no fork-only behaviour.
That decision changes the answers to most of this chunk.

**To confirm (2026-10-20):** with core changes made in ResourceSpace, the remaining question is whether
the plugin ships inside ResourceSpace (its yaml names Montala as author) or separately; shipping inside
fixes the version floor (Q0.5) by construction.

**Decided 2026-10-20: yes — the plugin ships inside ResourceSpace 11.1, as part of ResourceSpace.** Its
core changes land in the same release; there is no version floor, no patch list and no interim for
hook-less writes (I3 and C9 move to M3). The 11.1 release date is the plan's deadline.

## Done when

A policy per category is recorded; the patch list exists and is complete; each core bug has a
disposition; any PRs or issues are filed and referenced.
