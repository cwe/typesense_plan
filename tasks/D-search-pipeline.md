# D — Search pipeline

Milestone M2 builds D1–D11 to the point where a plain keyword search with every restriction is served.
The search forms themselves are the packages in [search/](search/README.md), cut to the catalogue; the
catalogue rows that belong to the pipeline (groups G, I and the request modes) are listed on the task
that owns them, and [search/coverage.md](search/coverage.md) has the full map.

### D1 — Hook entry
- **Rows:** G37 G38 G42 (the request modes; local harness only)
- **Builds:** `hooks/all.php` `HookTypesense_searchAllExternal_search(...)`, `include/search/run.php`.
- **Size:** S · **Depends on:** A1, A2
- **Must honour:**
  - The hook fires **before** `search_special()` and the standard query, so the plugin sees every
    search and must return `false` for anything it does not serve. Core's return handling needs the
    fork patch (I1).
  - Always fall back for `returnsql`, disk usage, `editable_only` and `smartsearch` (Q6.4 may change
    `editable_only` later).
  - Parameter types must accept what core passes: `csv_export_results_metadata.php` passes `null`
    for `$smartsearch` (a TypeError today for every group, even with the plugin disabled). Audit
    every parameter against every call site.
  - The "served" badge flag is set only for the main grid (array `fetchrows`, not refs-only); the
    selection-collection search on the same page once set it falsely.
  - The master toggle, the per-group override, the ready flag and the breaker all decide fallback
    before any work; Typesense-only mode per Q0.6 / Q6.5.
- **Blocking questions:** Q0.6, Q6.3, Q6.5
- **Proves it:** a hook battery: each request mode falls back; a null parameter is accepted; the badge
  flag only on the grid search; the breaker open → fallback.

### D2 — Context and parse
- **Rows:** I1 I4 I5 G41 · the special-argument forms F41 F68 F69 J3 J4 and the colon forms I6 I7 (SB1) are cases here too
- **Builds:** `include/search/parse.php` — the context (hook arguments and the user snapshot) and the parse.
- **Size:** M · **Depends on:** D1
- **Must honour:**
  - `$keywords` arrives post-processed by core: non-field colon tokens are split and appended
    (`12:30` → `12`, `30`); `@@` tokens are already resolved into `$node_bucket` /
    `$node_bucket_not`; `$archive` may hold a stray `""` (`is_int_loose` filter); `$order_by` is a
    resolved SQL fragment, not a key; `$search` still carries the special command and typed
    `field:value` text.
  - Core's special match is case-sensitive (`!LAST100` is an unknown special) and each special
    parses its argument differently (space vs comma, digits from the whole token); Q4.3 decides
    strict-and-veto vs replicate.
  - The search bar rebuilds node searches into text (`"field:Two Words"`, `subject:Birds`); the
    advanced form puts `!properties` before `!list`.
  - A single punctuated token (`foo-bar`) is a phrase in core when there is no plain-split path.
  - `$ignore_filters` reaches the hook but core uses it only in keyword parsing: it skips the date,
    `numrange` and fixed-list conversion of field terms (G41), so with it set a `date:2024` token is
    plain words — the parse must read the flag the same way (Q3.15).
- **Blocking questions:** Q4.3, Q3.2, Q2.8, Q3.15
- **Proves it:** a parse table covering catalogue groups I and J and every special's argument forms.

### D3 — Query plan and compiler
- **Builds:** `include/search/plan.php` — filter tree (AND groups, OR groups, join filters), sort list,
  `q`/`query_by`/weights, per-field `prefix`, `num_typos`, `drop_tokens_threshold=0`,
  `split_join_tokens`, candidate caps, offset/limit/pages, recent selection, result limit,
  suppression flags, open-access select, `supported`/`fallback_reason`; `compile()`; an
  operations counter.
- **Size:** M · **Depends on:** D2, D12
- **Must honour:**
  - `filter_by` is limited to 100 operations (operands + connectors; array values free; 50 clauses);
    a reference-join clause counts as one operand and its inner expression is costed separately
    (P34); an over-limit join expression ANDed with other clauses returns 0 **silently**, so the
    guard must veto first.
  - A compiled GET over 4,000 bytes must go as POST `multi_search`.
  - `prefix` must be explicit per field (`false` unless a wildcard): left unset, Typesense
    prefix-matches the last word of every search (`land` 3,172 / 69,408). `split_join_tokens`
    defaults to a retry that joins or splits words; `max_candidates` (10) and
    `max_filter_by_candidates` (4) cap wildcard completions.
  - Three sort fields maximum; omit `query_by` entirely for a match-all query.
  - **The composition rule (Q0.13):** the plan admits AND clauses plus five declared interaction
    mechanisms — the recent-N cutoff, restriction suppression by a mode, the `J`-inside-`!collection`
    rule, OR alternatives (a filter `||` group, or per-alternative searches merged in D10), and the
    budgets (the operations counter and a sort-slot check) — and nothing else; a form needing a new
    kind of interaction vetoes until its mechanism is designed with cases in SH1.
- **Blocking questions:** Q2.2, Q2.3, Q3.9, Q3.10
- **Proves it:** compiler battery against hand-written filters, the operations counter at the
  boundary, GET/POST choice, parameter defaults asserted on every compiled query.

### D4 — Shared keyword and node step (minimal)
- **Rows:** none of its own — SA1 (plain words), SC1 (node tokens) and SC2 (resolved options) own the
  M2 rows and are built on this step; the field-token split hands off to SB1.
- **Builds:** `include/search/criteria.php` — the shared step: plain words, wildcard, the field-token
  split that hands off to the field packages, node buckets, `query_by` from visible indexed fields with the title gated on
  `metadata_field_view_access`.
- **Size:** M · **Depends on:** D3, B1
- **Must honour:**
  - Keywords AND (`drop_tokens_threshold=0`); fixed-list `field:value` tokens already resolved into
    the bucket are dropped from `q`; an incidental colon is a space; a trailing `*` → prefix on that
    word, moved last (a per-field prefix list with more than two fields gave HTTP 400; use one
    boolean); two wildcards veto; negative field search, OR-group and full-text (`@FULL_TEXT`)
    veto (P30, P4) until area F decides otherwise.
  - Node buckets: AND across buckets, OR within (`nodes:=[…]`), `nodes:!=[…]` is any-of like core's
    `NOT EXISTS`; `$category_tree_search_use_and_logic` → one clause per node (untested: needs a
    tree fixture).
  - Only visible, active, indexed fields in `query_by`.
- **Blocking questions:** Q2.1, Q2.3, Q3.9
- **Proves it:** keyword battery (plain, wildcard, NOT, the vetoes) and node battery (OR bucket, AND
  buckets, exclusion, 600/1,000-id buckets, the 40/41-node boundary) against core.

### D5 — Standard restrictions
- **Rows:** G1–G10 G34 G35 G36 G40 H9 H10 (F20 F21 H38 belong to SF2: types and states are ignored for collections)
- **Builds:** `include/search/restrictions/standard.php`.
- **Size:** M · **Depends on:** D3
- **Must honour:** every rule of core's `search_filter()`:
  - Resource types: `restypes` CSV → `resource_type:=[…]`, skipped when blank, starting with
    `Global`, or for `!collection`; a list with only non-numeric entries means no resources in core;
    `T<type>` exclusions; specials per Q4.4.
  - Archive states: explicit list (`is_int_loose`), else `get_default_search_states()`; the
    per-special overrides; `$search_all_workflow_states`; `z<state>` and
    `$additional_archive_states` with `$uploader_view_override`; the pending rule as one OR group
    (`archive:!=[-2,-1] || created_by:=U || resource_type:=[ert…]`), not applied for `!collection`
    with `$k` and `$collection_allow_not_approved_share`.
  - `resource_created_by_filter` (`−1` = current user); the day limit on `creation_date` with core's
    cut-off `strtotime('today −n days')`; `ref:>0`; `$heightmin` → veto unless replicated.
  - `$ignore_filters` does **not** disable restrictions (core uses it only in keyword parsing).
  - `$smartsearch` keeps the saved archive states instead of the defaults (G40): the old plugin always
    fell back for it; serving it is the archive rule with the flag honoured (Q5.11).
- **Blocking questions:** Q4.4, Q4.5, Q6.3, Q6.6, Q5.11, Q6.9
- **Proves it:** restriction battery plus a config matrix (`$search_all_workflow_states`,
  `$collections_omit_archived`, `$uploader_view_override`, `$special_search_honors_restypes`).

### D6 — Access restriction
- **Builds:** `include/search/restrictions/access.php`.
- **Size:** S · **Depends on:** D3, B6
- **Must honour:**
  - For non-`v` users without `access_override`: `access:!=2 || $grants((user:=U && (expires:=0 ||
    expires:>now)) || usergroup:=G)` and `access:!=3 || $grants(usergroup:=G)` (P1).
  - `$access` for `v` users only, numeric only, regardless of `access_override` (P25).
  - Suppressed entirely for `!contributions<self>` with `$open_access_for_contributor` (P26).
- **Blocking questions:** — (confirm P1, P25, P26)
- **Proves it:** the 12-case access battery; live: 73 confidential resources hidden from an ungranted
  user, visible to a granted one.

### D7 — Group search filter
- **Builds:** `include/search/restrictions/group_filter.php`.
- **Size:** S · **Depends on:** D3
- **Must honour:**
  - `get_filter()` / `get_filter_rules()`: `nodes_on` → `nodes:=`, `nodes_off` → `nodes:!=`, inverted
    for NONE, rules ANDed (ALL/NONE) or ORed (ANY), the whole OR'd with a grant-exists clause when
    `$custom_access_overrides_search_filter` (**honouring grant expiry** — the old code ignored it)
    and with `created_by:=U` when `$open_access_for_contributor`.
  - Applied under `access_override` too (core applies it regardless).
  - `get_resource_access()` runs `!resource<ref>` for users with a filter, so this restriction decides
    per-resource access for them (see C8, Q6.2).
- **Blocking questions:** Q6.3
- **Proves it:** group-filter battery (ALL, NONE, ANY, override with an expired grant).

### D8 — Featured collections (`J`)
- **Builds:** `include/search/restrictions/featured.php`.
- **Size:** S · **Depends on:** D3, B5
- **Must honour:**
  - `compute_featured_collections_access_control()` may return `false` (no `j` permissions) → no
    results, never a TypeError.
  - Join filter `$memberships(collection_ref:=[…])`; with `j*` and no `-j`, core joins any collection
    (`collection_type:>=0`, 104,753 resources live) (P23).
  - Inside `!collection<C>`: two memberships clauses must match the same document, so skip the join
    when C is permitted, otherwise fall back; applies under `access_override`; the user's upload
    collection is exempt. Reuse candidate under principle 6: the old `featured_collections.php`
    rule (P23), reviewed.
- **Blocking questions:** Q5.8, Q6.3
- **Proves it:** the 11 `J` cases and 5 parse cases.

### D9 — Sort mapping
- **Rows:** G11 G14 G18 G22 G24 G25 G28 G29 G30 (served, or falling back to relevance as core does) · the
  declined sorts are packages SG1–SG5
- **Builds:** `include/search/sort.php`.
- **Size:** S · **Depends on:** D3, A3
- **Must honour:**
  - `order_by` fragments: `score …` → relevance; `field<date_field> …` → `date_field_sort`;
    `modified …` → `modified_date, ref`; `r.ref …` → `ref`; `c.sortorder …` → the collection mode's
    order; everything else vetoes (rating, popularity, colour, title, type, extension, status,
    random, custom fields) until Q7.2 adds sort keys.
  - Relevance on a match-all query is meaningless; the keyword-less order is Q7.1 — decided (B),
    2026-10-20: `hits_ur, date_ref` exactly, fresh through the hit-count hook (I3). **A keyword search
    ordered by relevance differs from core** (core's per-search sum of node hit counts cannot be
    reproduced): `_text_match` first, then `ur_hits, date_ref` — the one accepted difference under
    Q10.3, listed in the harness and in the user documentation (G4).
  - `!last` + relevance → `ref` with core's upper-case-DESC test (P29); a bare number with
    `$config_search_for_number` off → `_eval(ref:=N):desc` first (verified) if Q7.4 says so.
  - Modified ties broken by ref (80–97 % of rows share a `modified_date`).
  - Typesense takes three sort fields and breaks a final tie by insertion order, and appends
    `_text_match` as the last field when fewer than three are given (docs 30.0). Core's relevance,
    popularity and rating orders have four or five keys, all in one direction, so they map to the four
    packed `int64` keys B2 builds (`ur_hits`, `hits_ur`, `date_ref`, `rating_ur` — the table is in
    SG2 and the schema register); a keyword search keeps `_text_match` first (the listed exception). A single-field sort matches MySQL's NULL placement with
    `field(missing_values: first)` ascending and `(missing_values: last)` descending; a composite
    encodes "no value" in each part instead.
- **Blocking questions:** Q7.1, Q7.2, Q7.4, Q7.8
- **Proves it:** sort battery on a ties fixture; live date and modified order equal to core both ways.

### D10 — Execute and result windows
- **Rows:** G31 G32 G33 J7
- **Builds:** `include/search/execute.php` — one window or a walk, the recent-N cutoff, the count-only
  form.
- **Size:** M · **Depends on:** D3, A2
- **Must honour:**
  - `fetchrows` `−1`, `N`, `[offset, limit]`, `[0,0]`; `per_page=0` returns `found` with no hits;
    windows of 250 via `multi_search` (≤ 50 per request), de-duplicated; above
    `$typesense_search_max_rows` (25,000) veto (P22). Cost: 25k 1.5–4 s, 50k 10 s, whole index
    20–37 s; keyset on `ref` only for numeric sorts.
  - The search page passes `[offset − colcount, n]` when collections appear above results (Q7.5).
  - Recent-N: the Nth-highest ref via one extra query, then `ref:>=cutoff`. Reuse candidate under
    principle 6: the old `typesense_search_recent_cutoff()`, reviewed — it was live-validated.
  - OR alternatives, route B (Q2.7): one search per alternative from the same compiled plan in one
    `multi_search`, refs and sort keys collected to the row cap, de-duplicated, ordered by the
    requested sort (relevance approximate), then windowed; the total is the union's size.
- **Blocking questions:** Q7.3, Q7.5, Q7.7, Q2.7
- **Proves it:** the 40-check window battery (rewritten into the harness), including the 250-row
  boundary with ties.

### D11 — Hydrate
- **Rows:** G42 (the result shapes; local harness only)
- **Builds:** `include/search/hydrate.php` — rows from MySQL in Typesense order, shapes, validation
  hook (C8).
- **Size:** M · **Depends on:** D10
- **Must honour:**
  - `ORDER BY FIELD(r.ref, …)` in batches of ~1,000 (two placeholders per ref; MySQL's 65,535 cap at
    ~32k refs); core's `$select` plus the `rca`/`rca2` joins when it references them; the
    node-bucket case appends `total_hit_count` aggregates to `$select` (hydrate crashed until the
    joins were reproduced); `!contributions` open access uses a zeroed select.
  - Shapes: refs-only `[{ref}]`; structured `['total','data']` for array `fetchrows`; int-`N`
    zero-padded to the total like core; keep the total when refs are empty (`[0,0]` counts were 0);
    `csv_export_results_metadata` reads a `fetchrows=1` result to get the total.
  - `!collection` rows need `c.date_added`, `c.comment`, `commentset` (Q5.2).
- **Blocking questions:** Q6.1, Q5.2, Q7.6
- **Proves it:** a shape battery (every `fetchrows` shape vs core), the node-bucket hydrate case,
  then C8's stale-index battery once validation is in.

### D12 — Escaping and the injection battery
- **Rows:** none of its own — every value in every row passes through it.
- **Builds:** `include/search/escape.php` — `filter_value(string)` (backtick-quoted, with the rule
  for a value that itself contains a backtick), `filter_int_list(array)`, `q_value(string)` (what
  may stay in `q`: `:`, `-`, `"` and `*` carry meaning there), `sort_eval_int(int)`, and the rule
  that hydrate's ref lists are `ps_query` placeholders, never interpolated; used by D3, D4, D9, D11
  and every search package.
- **Size:** S · **Depends on:** D2 (D3 depends on this)
- **Must honour:**
  - Everything in the hook's `$search` is user input end to end: field values, quoted phrases,
    `!list` refs, `!collection` ids, `@@` tokens, `numrange` bounds, date-range bounds; integers are
    cast, names come from the lookup, never from the string.
  - `&&`, `||`, `:=`, `:!=`, `[`, `]`, `(`, `)` inside a backtick-quoted value are literal in
    Typesense — proven on the local server for every operator the plan uses, not assumed; a value
    with a backtick cannot be quoted: reject (veto) or strip, decided once.
  - `_eval(ref:=N)` takes an int only; `filter_by` field names such as `populated_field_ids:=F`
    take ints only.
  - A compiled query the server rejects (HTTP 400) becomes a veto, never an empty result.
- **Blocking questions:** — (Q6.3 if a reject rule is preferred to stripping)
- **Proves it:** `harness/ab/D12_escaping.php` — a fuzz list (backticks, quotes, colons, `&&`, `||`,
  brackets, `*`, `-`, newlines, NUL, a 4,000-byte value, SQL fragments) through every form that
  takes a value, asserting the compiled parameters are what the escaper says, that Typesense returns
  the expected documents or a 400 the plugin turns into a veto, and that no value can widen a
  `filter_by` — the live user cannot see a document the restriction excludes.
