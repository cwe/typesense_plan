# Chunk 4 — Special searches

Status: planning. Depends on: 0, 1. Feeds: 10.

## Scope

The `!command` searches: which are served, which stay on core, how arguments are parsed, how
`restypes` and workflow states apply to them, and the smart-collection side effect.

## What we know

- Served: `!collection` (chunk 5), `!last` (newest N, then the user's sort; non-numeric argument →
  1000; relevance → ref order with core's upper-case-DESC test), `!list`/`!listall`, `!resource` and
  bare numbers with `$config_search_for_number`, `!contributions` (open-access replicated),
  `!hasdata` (works for non-indexed fields via `populated_field_ids`), `!archivepending`,
  `!userpending`.
- Vetoed to core: `!properties`, `!images`, `!nopreview`, `!geo`, `!colour`, `!colourkey`, `!rgb`,
  `!related`, `!relatedpushed`, `!duplicates`, `!nodownloads`, `!integrityfail`, `!locked`,
  `!noningested`, `!report`, `!unused`, `!empty`. Those marked in chunk 1 need new index fields;
  `!rgb` (computed colour distance), `!duplicates` (GROUP BY HAVING), `!nodownloads` (activity),
  `!report` (arbitrary SQL) cannot be served.
- `!empty<field>` is expressible as `populated_field_ids:!=[ref]` (verified: 112,596 = total − 12),
  but core's `!empty` is buggy: a field *name* binds as an integer so any name means field 1 (Core 5),
  and a non-global field's type condition sits inside the NOT IN (Core 6).
- Parsing differences: core's command match is case-sensitive, so `!LAST100` is an unknown special
  and returns **everything** (as does any misspelt special); the plugin parses commands
  case-insensitively and serves `!LAST100` as `!last1000` (H7). `!lastabc` (H6) and a second special
  word (H8) differ. `!resource416, date:2024` looks up resource 4162024 in core (Core 3); the plugin
  takes the first token's digits. A space after `!last100` makes it 1000, a comma keeps 100 (Core 7).
- `restypes`: core's `search_filter` applies them to every search except `!collection`; it is the
  search page that blanks them for specials. The plugin honours them on specials only with
  `$special_search_honors_restypes`, so API and internal callers differ (review item 5).
- Workflow states: `!collection`, `!list` and the two pending searches ignore the state parameter;
  `!related`, `!hasdata`, `!integrityfail` search every state; the rest honour it. `!archivepending`
  returns nothing for a user with `z1`.
- Smart collections: the plugin refreshes before it knows it will serve the search (a later veto makes
  core refresh again) and lacks core's 2-hour rate limit for featured smart collections.
- Core bug: `!last` + a node bucket returns one row (Core 1). `!related<ref> <term>` timed out at
  120 s in core on the test system.
- The advanced search form puts `!properties` in front of `!list`, so with any property filled in the
  resource-IDs box is ignored by core (J1/J2).

## Candidate work items

- Modes for the specials chosen in Q4.1, with their index fields from Q1.6.
- Argument parsing aligned with the decision in Q4.3.
- `restypes` and workflow-state handling per special, matching the decided source of truth.
- Move the smart-collection refresh to after the plugin commits to serving; add the rate limit.

## Questions to answer before coding

### Q4.1 — Which vetoed specials to serve, in what order?
Earlier recommendation: `!related` first, then `!properties`, then the flag-based ones (`!images`,
`!nopreview`, `!locked`, `!integrityfail`, `!noningested`), then `!geo`, `!colour`/`!colourkey`,
`!unused`, `!empty`. Confirm the order, and confirm `!rgb`, `!duplicates`, `!nodownloads`, `!report`
stay on core permanently.

### Q4.2 — Index fields for them: one batch or per special?
Each new special needs fields (chunk 1, Q1.6) and a rebuild. One schema change and one rebuild for
all of them, or incremental?

### Q4.3 — Argument parsing: replicate core exactly, or parse strictly and veto?
Core's parsing is inconsistent (space vs comma, digits from the whole token, case-sensitive
commands, unknown special → everything). Options: (a) replicate it, bugs included, so results match;
(b) parse each special's documented form and veto anything else (core then answers — and returns
everything for an unknown special, so the user sees core's behaviour either way); (c) parse
sensibly and accept differences. Recommendation: (b).

### Q4.4 — `restypes` on specials
Follow `search_filter` (apply to all but `!collection`) so API callers match core, or follow the
search page's blanking (current config-gated behaviour)? The two cannot both be matched.

### Q4.5 — Workflow states per special
Replicate core's per-special table exactly, including `!archivepending` returning nothing for `z1`
users? Or decline the two pending searches for users who cannot see those states?

### Q4.6 — Smart collections
Move the refresh after the serve decision and add the 2-hour limit, or veto smart collections to
core altogether (simplest, and `$allow_smart_collections` is off by default)?

### Q4.7 — `!unused` (resources in no collection)
Needs absence-of-match against the memberships collection. Verify Typesense can express it
(a negated reference join) before deciding; otherwise it stays on core.

### Q4.8 — `!empty<field>`
Serve it correctly from `populated_field_ids` (diverging from core's bug), replicate the bug, or
veto? (Links Q11.3.)

### Q4.9 — Unknown or misspelt specials
Core returns everything. The plugin vetoes, so the user gets core's behaviour. Keep, or make the
plugin return nothing / a declined-search message for `!unknown` (a deliberate difference)?

### Q4.10 — The form's `!properties` + `!list` collision
Core ignores the resource-IDs box when a property is filled in. Replicate, or raise with core?

## Done when

Catalogue group F has a decided behaviour per row and a harness case; each served special has a mode
with its own battery; the vetoed list is documented with the reason for each.
