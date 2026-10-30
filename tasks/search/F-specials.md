# SF — Special searches (catalogue group F)

A search starting with `!` is handled by `search_special()` (or by the keyword stage for `!empty`).
Anything after the first space is keywords, nodes or field terms applied on top; a second word
starting with `!` is ignored. Resource types still apply to most of them, workflow states do not
apply to collections, lists and the pending searches, and the `z` permissions always apply.

**Catalogue marks (2026-10-16):** required — SF1–SF8 (F40, F41 and F44 not required but served by
composition anyway), SF9 `!properties`, SF10 `!empty` (not the name form), SF11 `!related`, SF12
`!images`, SF13 `!locked` (`!integrityfail`, `!noningested` to check); *check* — SF16 `!unused`;
parked as not required — SF14 `!geo`, SF15 `!colour`; SF17 confirmed on core. Rows marked *check*:
F31, F37, F38, F39, F48, F52, F59, F62, F63, F65, F69.

**Catalogue marks (2026-10-30):** the check rows are decided — F37, F39, F48, F52, F59, F62 and F69 *must*; F63
and F65 *no* (SF13 keeps only `!locked`); F31 a bug row. SF16 is required, blocked by Q4.7 and Q5.7.

One package per special or family. SF1–SF8 are milestone M5 (the data is indexed already);
SF9–SF16 are M7, each needing its index fields (Q4.1 orders them, Q4.2 batches the schema change);
SF17 is the list that stays on core. Every mode gets keyword, node and field matching from D4, so
`!x … sculpture @@389 title:park` composes without extra work — each battery has one such case.

### SF1 — `!last`
- **Rows:** F1 F2 F3 F4 F5 F69 H12 · cross: F68 (SF8), I9 (SF8)
- **Builds:** `modes/last.php` — the recent-N selection (D10) with the argument rule.
- **Size:** S · **Depends on:** D10, D9 · **Milestone:** M5
- **Core:** the N highest refs that pass the filters (inner `ORDER BY ref DESC LIMIT n`), then
  the outer `ORDER BY` of the user's sort; relevance becomes `ref DESC` (F2, P29); the number is
  read from everything after `!last` up to the first comma, so `!last100 sculpture` is "100
  sculpture" → not an integer → 1000 (F4, P28) while `!last100, sculpture` keeps 100 (F69);
  `!lastabc` is 1000 (F3); `!last1000 @@389` returns one row — an aggregate without GROUP BY
  (F5, Core 1).
- **Serve:** `setRecentSelection(N)`: one extra query for the Nth-highest ref among the matches,
  then `ref:>=cutoff` and the user's sort (verified live both directions); keyword, node and
  field criteria apply inside the selection; the node-bucket case served correctly, replicated or
  vetoed per Q11.3.
- **Must honour:** the home view is `!last1000` (`header.php`), so this mode carries the main
  grid's sort dropdown; the upper-case-DESC test for relevance (P29); the comma form; a
  non-numeric argument means 1000, not a veto.
- **Battery:** `harness/ab/SF1_last.php` — F1–F5, F69, H12, the 11 existing last cases, D16/D16b,
  16_core_bugs `!last + node`.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** `!lastN, word`: the keyword after the first space applies, `N` from `explode(',')[0]` (`sf:1127-1135`), proved live (`!last100, sculptre` = 1); through the API the hundred are listed ascending (P29's upper-case `DESC` test).
- **Blocking questions:** Q4.3, Q11.3

### SF2 — `!collection`
- **Rows:** F9 F10 F11 F12 F13 F14 F15 F16 F17 F18 F19 F20 F21 H11 H26 H38 · cross: I1 (D2: keywords only after the first space)
- **Builds:** `modes/collection.php`, `include/collections.php` (the access gate: core helper or
  plugin copy, Q5.1 / I2).
- **Size:** M · **Depends on:** D4, D8, B5 · **Milestone:** M5
- **Core:** members of a readable collection in any workflow state (`z` permissions still
  apply, F21); readable = own, public (F10), featured through
  `featured_collection_check_access_control` (F11, F12), request, research, upload-share session
  collections, the share key, `$ignore_collection_access`, then `collection_readable()`; someone
  else's private collection is empty (F13); a negative ref is the user's upload collection (F15);
  the collection id is split on space, then comma, then `(int)` (I1: `!collection3808,sculpture`
  drops the keyword; F17 `!collection3808, sculpture` keeps it); resource types are **not** applied
  (F20, H38); collection order is `c.sortorder, c.date_added reversed, ref` (F19, P18); keywords,
  nodes and dates apply inside (F16, F18, H11, H26).
- **Serve:** `$memberships(collection_ref:=C)`; default-archive restriction suppressed;
  `archive:!=2` when `$collections_omit_archived` and no `e2`, only without `access_override`; the
  `$k` pending exemption (D5); default order `$memberships(sortorder:d, date_added:opposite),ref:d`
  (verified on three collections both ways); an explicit sort through D9; volatile collections
  (selection, upload, share-upload) veto (P27); smart collections per Q4.6; `J` inside a
  collection per P23 (D8).
- **Must honour:** the gate is never skipped under `access_override`; the collection rows need
  `c.date_added`, `c.comment`, `commentset` at hydrate (Q5.2); `!collection158` errors in core on
  the test system — unexplained; the selection bar's search is a volatile collection and once set
  the "served" badge falsely (D1).
- **Fixture:** the base fixture's collections (own, public, featured, another user's private, an
  upload collection, a share key, an upload-share session).
- **Battery:** `harness/ab/SF2_collection.php` — the rows above, the 13 collection cases, F30–F32
  (R/h users, access_override), T1/T2 (upload-share gate), the parity plan's Phase 2 matrix
  (users × collections × queries × `fetchrows`) live.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** `!collection` skips `resource_type IN` and the archive parameter but keeps the pending and z filters (`sf:751`, `825-902`); collection order is `c.sortorder asc, c.date_added DESC, r.ref asc` (`sf:3196-3207`); collections of 577,722 members with sort orders exist on the second database.
- **Blocking questions:** Q5.1, Q5.2, Q5.3, Q5.4, Q5.8, Q4.6, Q5.9, Q5.10, Q6.8

### SF3 — `!list` and `!listall`
- **Rows:** F38 F39 F40 F41 H28 · cross: J1 (SJ1), F41 (D2's parse table)
- **Builds:** `modes/list.php`.
- **Size:** S · **Depends on:** D4 · **Milestone:** M5
- **Core:** colon-separated refs from the first token (`!list416,4460` is resource 416 only,
  F41); non-integers dropped; none valid → nothing (`ref < 0`); workflow states not applied, `z`
  permissions are; `!listall` is identical in core (the difference is in `search_filter`, which
  treats both alike); keywords and nodes apply (F40, H28); the advanced form sends the resource IDs
  box as `!list…` followed by its other terms (J1).
- **Serve:** `ref:=[…]`, every archive state, D5's `z` exclusions kept.
- **Battery:** `harness/ab/SF3_list.php` — F38–F41, H28, J1 and the parse cases.
- **Blocking questions:** Q4.3
- **Catalogue marks (2026-10-20):** F38 *must* — the advanced form's J1 (SJ1) depends on `!list`, so the
  package is required; F39 and H28 *check* (the state rule and the node AND, on the fixture: the live refs
  are hidden from the test users); F40 and F41 *no*, served by composition, F41 being Q4.3's parse rule.

### SF4 — `!resource` and bare numbers
- **Rows:** F30 F31 · cross: A25 (SA9)
- **Builds:** `modes/resource.php`; the `$config_search_for_number` branch.
- **Size:** S · **Depends on:** D4, D9 · **Milestone:** M5
- **Core:** one resource by ref (the search bar's resource ID box); **every digit in the string**
  forms the ref, so `!resource416, date:2024` looks up 4162024 and finds nothing (F31, Core 3);
  with `$config_search_for_number` a bare number is the same search, otherwise it is a word plus
  the exact ref ordered first (A25).
- **Serve:** `ref:=N` from the first token's digits, or replicate the all-digits rule per Q4.3;
  `get_resource_access()` runs this search as a gate for users with a search filter (Q6.2), so a
  stale index must not answer it (C8).
- **Battery:** `harness/ab/SF4_resource.php` — F30, F31, A25 both settings, R1/R4 (a special
  followed directly by a term with digits), the gate case from C7.
- **Blocking questions:** Q4.3, Q6.2, Q7.4

### SF5 — `!contributions`
- **Rows:** F34 F35 H27 · cross: J3 J4 (SJ1)
- **Builds:** `modes/contributions.php`.
- **Size:** S · **Depends on:** D4, D6 · **Milestone:** M5
- **Core:** `created_by = U`; for the user's own with `$open_access_for_contributor`, core drops
  every filter and join (keyword criteria included) and zeroes the grant columns (F35); the user
  actions page uses it for submit-pending resources (state −2); the advanced form posts it with a
  comma and no space, so the following term is dropped (J3) and only what follows the first space
  survives (J4 — `title:park` lost, `sculpture` applied).
- **Serve:** `created_by:=U`; the open-access form replicated (P26): suppress all restrictions and
  keyword matching, `ref:>0`, `archive:=[raw states cast to int, "" → 0]`, veto on an empty list,
  zeroed select at hydrate; `restypes` per Q4.4 (harness D35: 5 / 29 when the old plugin ignored
  them).
- **Battery:** `harness/ab/SF5_contributions.php` — F34, F35, H27, J3, J4, the 17 contributions
  cases, D35.
- **Blocking questions:** Q4.4 (confirm P26)

### SF6 — `!hasdata`
- **Rows:** F42 F43 F44 H29
- **Builds:** `modes/hasdata.php`.
- **Size:** S · **Depends on:** D4 · **Milestone:** M5
- **Core:** resources with any value in the field, by ref only; every workflow state; a node
  join, so indexing does not matter (F43); keywords apply (F44).
- **Serve:** `populated_field_ids:=[ref]`; works for non-indexed, inactive and hidden fields
  because `populated_field_ids` comes from all nodes (B3); `restypes` per Q4.4.
- **Battery:** `harness/ab/SF6_hasdata.php` — F42–F44, H29, harness 21 (non-indexed, inactive,
  hidden field), R4.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** `!hasdata` has no archive clause; pending and z filters and restypes apply; the node join reduces the set (12 → 5 on the test system).
- **Blocking questions:** Q4.4

### SF7 — `!archivepending` and `!userpending`
- **Rows:** F32 F33
- **Builds:** `modes/pending.php`.
- **Size:** S · **Depends on:** D4, D5 · **Milestone:** M5
- **Core:** state 1 and state −1; the state parameter is ignored; `!archivepending` returns
  nothing for a user with `z1`; `!userpending` shows non-admins only their own and, with the
  rating sort, orders by `request_count`.
- **Serve:** `archive:=1` / `archive:=-1` with D5's `z` exclusions and the own-resources rule;
  rating sort → veto (D9).
- **Battery:** `harness/ab/SF7_pending.php` — F32, F33 for a `z1` user and a `v` user.
- **Blocking questions:** Q4.5

### SF8 — Unknown, misspelt and doubled specials
- **Rows:** F67 F68 I9
- **Builds:** `modes/unsupported.php` — the catch-all veto; the first-token rule.
- **Size:** S · **Depends on:** D2 · **Milestone:** M2 (it is what makes every other `!` safe)
- **Core:** an unknown special has no branch: a plain search with no terms, so **everything**
  (F67); the prefixes are compared case-sensitively, so `!LAST100` is unknown too (I9); a second
  `!` word is skipped by the keyword stage, and the space makes the first's number invalid
  (`!last100 !collection3808` = `!last1000`, F68).
- **Serve:** veto every `!command` no other mode claims — the user then gets core's everything,
  which Q4.9 may want to change to a declined-search message; case-sensitive matching as core;
  only the first `!` token is a command.
- **Battery:** `harness/ab/SF8_unknown_specials.php` — F67, F68, I9, every row of group F not
  served at the current milestone declines with a reason.
- **Blocking questions:** Q4.9

### SF9 — `!properties`
- **Rows:** F50 F51 F52 F53 F54 F55 F56 F57 F58 F59 F60 F61 F62 H30 · cross: J2 J11 (SJ1)
- **Builds:** `modes/properties.php`; index fields `file_extension`, `file_size`, `width`,
  `height`, `has_image`, `created_by` (exists), an orientation derived at index time (or
  computed from width/height in the filter).
- **Size:** M · **Depends on:** D4, B2 (registry entries), B8 (a rebuild), C5 (E13, E14) · **Milestone:** M7
- **Core:** `!properties` followed by `;`-separated `key:value` pairs, ANDed: `fext:jpg`
  (`LIKE "jpg"`), `fext:-jpg` (`NOT LIKE`), `fext:jp*` (`*` → `%`), `hmin`/`hmax`/`wmin`/`wmax`
  on `resource_dimensions`, `fmin`/`fmax` in MB on `file_size`, `pi:1` has a preview, `cu:9`
  contributor, `orientation:portrait|landscape|square` from the dimensions, `i`/`s` keys also
  exist in the code; an unknown key adds no filter: everything (F62); keywords and nodes apply
  after a space (F60, F61); the advanced form puts `!properties` **first**, joined to the rest with
  " ," — so with any property filled in the resource IDs box (`!list`) becomes a second special
  and is ignored (J2, Q4.10) and the terms after the separator apply (J11).
- **Serve:** one filter per key on the new fields; `fext` wildcard as a prefix match on
  `file_extension`; orientation from stored width/height; unknown key per Q4.11 (nothing, as core,
  or veto).
- **Must honour:** the full key list is in `search_special()` (`cu fext fmax fmin hmax hmin i
  orientation pi s wmax wmin`); `file_size` in bytes vs the form's MB; the collision with `!list`.
- **Fixture:** resources with extension, size, dimensions and preview flags set.
- **Battery:** `harness/ab/SF9_properties.php` — F50–F62, H30, J2, J11.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** `fext`: `*` → `%`, a leading `-` → `NOT`, a leading `.` stripped, `LIKE` in the column collation (`sf:1566-1578`) — `jp*` is jpg, jpeg and jpf on the test system; `orientation`: `COALESCE(height, 0) > COALESCE(width, 0)` through a LEFT JOIN on `resource_dimensions`, so a resource with no dimensions row is square (`sf:1588-1606`); an unknown property name adds no filter (`sf:1539`).
- **Blocking questions:** Q4.1, Q4.2 (both settled 2026-10-30: the marks, and Q1.6 for the fields), Q4.10
- **Open questions:** Q4.11 (the key set and the unknown-key rule) — decided 2026-10-30

### SF10 — `!empty`
- **Rows:** F45 F46 F47 F48 F49
- **Builds:** `modes/empty.php`.
- **Size:** S · **Depends on:** D4, D5 · **Milestone:** M7
- **Core:** resources with no value in the field, handled in the keyword stage; **the name form
  binds as an integer** so `!emptynumberfield` and `!emptynosuchfield` both become `!empty1`
  (F46, F49, Core 5); the resource-type condition sits inside the NOT IN, so `!empty95` returns
  resources of every other type too (F47, Core 6); `restypes` limit the candidates (F48).
- **Serve:** `populated_field_ids:!=[ref]` (verified: 112,596 = total − 12) with the field's
  resource types applied correctly, or replicate the bugs, or veto (Q4.8 / Q11.3); the name form
  resolved through SB1's lookup.
- **Battery:** `harness/ab/SF10_empty.php` — F45–F49, harness I7.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** `!empty` with restypes: the inner type clause is correlated and inert (F47), and `search_filter()`'s `resource_type IN` limits the result (`dsk:551-568`, `sf:751-758`); a non-global field with no type overlap produces `IN ()`, a SQL error (Q11.3).
- **Blocking questions:** Q4.8, Q11.3

### SF11 — `!related` and `!relatedpushed`
- **Rows:** F22 F23 F24
- **Builds:** `modes/related.php`; a `related[]` field (both directions) on the resource.
- **Size:** M · **Depends on:** D4, B2 (a `related` entry), C9 (E18: no hook today) · **Milestone:** M7
- **Core:** both directions of `resource_related`, all workflow states, `$related_search_show_self`;
  `!related121409 sculpture` ran past the 120-second API timeout on the test system (F23);
  `!relatedpushed` limits to resource types with `push_metadata` (the view page).
- **Serve:** `related:=[ref]` from the field; `!relatedpushed` adds `resource_type:=[push types]`;
  freshness only through C9's `after_related_resources_changed` — `update_related_resource()` fires
  no hook today (E18), so until I3 lands the field is fresh nightly.
- **Battery:** `harness/ab/SF11_related.php` — F22–F24 with a keyword, a node and a sort.
- **Blocking questions:** Q4.1, Q4.2 (both settled 2026-10-30: the marks, and Q1.6 for the fields)

### SF12 — `!images` and `!nopreview`
- **Rows:** F29 F36
- **Builds:** `modes/images.php`; `has_image` on the resource.
- **Size:** S · **Depends on:** B2 (a `has_image` entry), C5 (E13) · **Milestone:** M7
- **Core:** `has_image > 0` (F36 100,757) and `has_image = 0` (F29 105).
- **Serve:** `has_image:>0` / `has_image:=0`; freshness through `afterpreviewcreation` (C5, E13).
- **Battery:** `harness/ab/SF12_images.php` — F29, F36, with a keyword and restypes.
- **Blocking questions:** Q4.1, Q4.2 (both settled 2026-10-30: the marks, and Q1.6 for the fields)

### SF13 — `!locked`, `!integrityfail`, `!noningested`
- **Rows:** F63 F64 F65
- **Builds:** `modes/flags.php`; `lock_user`, `integrity_fail`, a file-present flag.
- **Size:** S · **Depends on:** B2 (three entries), C9 (E16: the lock has no hook) · **Milestone:** M7
- **Core:** `lock_user <> 0` (F64); `integrity_fail = 1` and a file present, all states (F63);
  `!noningested` is for admins (resources with a `file_path`) — for anyone else no branch matches
  and it is a plain search: everything (F65).
- **Serve:** `!locked` only. F63 and F65 are *no* (2026-10-30): a maintenance flag set outside the indexing
  pipeline, and F67's bug (the admin half applies no `$sql_filter`), so `!integrityfail` and `!noningested`
  decline to core.
- **Battery:** `harness/ab/SF13_flags.php` — F64 for an admin and a standard user; F63 and F65 assert the decline.
- **Code facts (2026-10-23, [checks-findings.md](checks-findings.md)):** `!integrityfail` matches the exact string only — with a keyword it silently runs as a plain search (`sf:1619`); `!noningested` for a non-admin falls through to everything, and for an admin runs with no `$sql_filter` at all (`sf:1629-1632`).
- **Blocking questions:** Q4.1, Q4.2 (both settled 2026-10-30: the marks, and Q1.6 for the fields)

### SF14 — `!geo`
- **Rows:** F25
- **Builds:** `modes/geo.php`; a geopoint field.
- **Size:** S · **Depends on:** B2 (a geopoint entry) · **Milestone:** M7
- **Core:** `!geo53p5bm1p7t53p7bm1p4`: a bounding box where `m` is minus, `p` the point, `b`
  separates latitude and longitude, `t` the two corners (53.5,−1.7 to 53.7,−1.4): 4,893.
- **Serve:** a Typesense geo polygon or bounding-box filter on the point.
- **Battery:** `harness/ab/SF14_geo.php` — F25, a box crossing 0° and the antimeridian.
- **Blocking questions:** Q4.1, Q4.2 (both settled 2026-10-30: the marks, and Q1.6 for the fields)

### SF15 — `!colour` and `!colourkey`
- **Rows:** F26 F27
- **Builds:** `modes/colour.php`; `colour_key` on the resource.
- **Size:** S · **Depends on:** B2 (a `colour_key` entry), C5 (E13) · **Milestone:** M7
- **Core:** `!colourkeyEPNB` is the first four letters of the colour key with a preview present
  (1,979); `!colourE` is `colour_key LIKE "E%" OR "_E%"` (70,574).
- **Serve:** prefix filters on `colour_key` (`E*`, and the second-character form needs either a
  second field or a token per position).
- **Battery:** `harness/ab/SF15_colour.php` — F26, F27.
- **Blocking questions:** Q4.1, Q4.2 (both settled 2026-10-30: the marks, and Q1.6 for the fields)

### SF16 — `!unused`
- **Rows:** F37
- **Builds:** `modes/unused.php` — resources in no collection.
- **Size:** S · **Depends on:** B5 · **Milestone:** M7
- **Core:** in no collection (6,967).
- **Serve:** absence of any membership — verify Typesense can express a negated reference join
  before deciding (Q4.7); otherwise a maintained `in_collections` count on the resource, or core.
- **Battery:** `harness/ab/SF16_unused.php` — F37, after an add and a remove (freshness).
- **Note (2026-10-29, F37 marked must):** core's `!unused` is `r.ref NOT IN (SELECT c.resource FROM
  collection_resource c)` with no join on `collection.type` (`search_functions.php:1471-1484`): a
  resource held only in a user's private (type 0), selection (2), upload (1), share-upload (5) or
  request (6) collection counts as used. Chris is not sure core should count the private and
  selection kinds — a question for core, since the documentation says "in no collection". The
  membership index excludes selection, upload and share-upload collections (P27), so an exact F37
  needs either every kind indexed (Q5.7) or core's definition settled first; until then the plugin
  cannot serve `!unused` identically and declines. Blocking question Q5.7 added.
- **Blocking questions:** Q4.7, Q5.7

### SF17 — Specials that stay on core
- **Rows:** F6 F7 F8 F28 F66
- **Builds:** entries in the unsupported list with their reasons; a doc line each.
- **Size:** S · **Depends on:** SF8 · **Milestone:** M5
- **Core:** `!nodownloads` (not in `daily_stat` as a download), `!duplicates` (every resource whose
  checksum appears more than once, or those sharing one resource's checksum), `!rgb` (nearest
  average colour, at most 500 rows), `!report` (a report's SQL as results, needs the permission).
- **Serve:** never — activity data, a GROUP BY … HAVING, a computed distance sort and arbitrary
  SQL have no Typesense equivalent worth building (Q4.1 confirms).
- **Battery:** in `SF8_unknown_specials.php` — each declines with its reason.
- **Blocking questions:** Q4.1
