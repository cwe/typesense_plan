# Decisions already taken (before the restart)

Each was taken in an earlier session and most are implemented. On the restart each needs a
**Confirm** or **Revisit** in the last column; a revisit becomes a question in the relevant chunk.

| # | Date | Decision | Where | Chunk | Confirm / revisit |
|---|---|---|---|---|---|
| P1 | 18 Sep | Access control via a separate grants collection joined at query time, expiry checked live (Option A) | architecture doc | 6 | |
| P2 | 18 Sep | Pipeline architecture: one mode + all restrictions + shared keyword step; hook-extensible registries | architecture doc | 0 | |
| P3 | 21 Sep | `!last` and `!collection` honour an explicit sort; collection order is the default only | sorting fixes | 7 | |
| P4 | 21 Sep | OR-groups and full-text boolean searches veto to core | query.php | 2 | |
| P5 | 23 Sep | Date-range `field:value` is an interval overlap on `_range_start`/`_range_end`, all date field types | query.php | 3 | |
| P6 | 23 Sep | Numeric single bound = exact match (core's `rnn.name = max(min,max)`) | query.php | 3 | |
| P7 | 23 Sep | Sync: nightly rebuild into generation collections behind aliases; old generation deleted after a verified swap | sync plan D1 | 8 | |
| P8 | 23 Sep | Sync: hooks write to a queue table; the request flushes its own rows before responding; per-minute cron worker drains the rest | sync plan D2, D8 | 8 | |
| P9 | 23 Sep | Sync: mid-build changes replayed from stamped queue rows, no dual-write | sync plan D3 | 8 | |
| P10 | 23 Sep | Sync: while bulk rows are pending, searches are served by MySQL; bulk path pipelined/parallel | sync plan D4, D9 | 8 | |
| P11 | 23 Sep | Sync: offline jobs only for admin rebuilds; fail if the build lock is held; no rescheduling | sync plan D5 | 8 | |
| P12 | 23 Sep | New core hooks are acceptable; the node-layer hook first | sync plan D6 | 8, 11 | |
| P13 | 23 Sep | Read-time validation stays, default on; plugin global (`disable_group_select`) with per-group enable override | sync plan D7 | 6, 8 | |
| P14 | 23 Sep | Deleted-state resources stay in the index with their archive value | sync plan D10 | 8 | |
| P15 | 23 Sep | Rebuild time = the administrator's scheduled-task entry; the plugin reports last run/result | sync plan D11 | 9 | |
| P16 | 23 Sep | Import concurrency, chunk size, worker count: safe defaults, configurable, tuned from the rebuild's timing report; halve concurrency on 503 | sync plan D12 | 9 | |
| P17 | 23 Sep | Collection freshness follows the resource rules (hooks + nightly rebuild); the per-collection fingerprint check is dropped | sync plan D13 | 5, 8 | |
| P18 | 24 Sep | NULL `sortorder` → INT32_MIN, NULL `date_added` → 0; collection order `sortorder, date_added reversed, ref` | collection fixes | 5 | |
| P19 | 24 Sep | Index `creation_date` (day limit) and `date_field_sort` text key (date sort); `created_date` removed; schema upgraded by PATCH at reindex | reindex date fix | 1, 7 | |
| P20 | 24 Sep | `created_by` optional; NULL indexed as no value | created_by fix | 1 | |
| P21 | 24 Sep | `reindex.php --drop` for testing; no search fallback while rebuilding under `--drop` | --drop | 9 | |
| P22 | 24–25 Sep | `$typesense_search_max_rows` (25,000) with window walking via `multi_search`; hydrate in batches | truncation fix | 7 | |
| P23 | 25 Sep | `J` inside a collection: skip the join when the collection is permitted, else fall back; `j*` with no `-j` replicates core's any-collection behaviour (not fixed in core) | J1 | 5 | |
| P24 | 25 Sep | "Rebuild every 15 min + in-memory flush, no queue" alternative declined | sync plan | 8 | |
| P25 | 29 Sep | `$access` filter for `v` users only, numeric only, regardless of `access_override` | $access fix | 6 | |
| P26 | 29 Sep | `!contributions<self>` with `$open_access_for_contributor` **replicated** (suppress all restrictions and keyword matching, zeroed grant columns), not vetoed | contributions | 4, 6 | |
| P27 | 29 Sep | Selection (type 2) and upload (type 1 / negative ref) collections are not indexed; collection mode vetoes them | collection fixes | 5 | |
| P28 | 29 Sep | `!last<n> <text>` with a space → 1000, as core | last.php | 4 | |
| P29 | 29 Sep | `!last` + relevance → ref order with core's upper-case-DESC test | last.php | 7 | |
| P30 | 29 Sep | Free-text wildcard: `prefix=true`, wildcarded word moved last, two wildcards veto | query.php | 2 | |
| P31 | 29 Sep | Stem every text field iff `$stemming`, `_q` never; `token_separators` = `$config_separators` + `,` `:` (single-byte only) | schema | 1 | |
| P32 | 29 Sep | Date `field:value` uses exact `field_<ref>_q:=<value>` | query.php | 3 | |
| P33 | 29 Sep | Strip invisible characters (`\p{Cf}`, `\p{Zs}` → space) at the indexer's text sites and typed values | clean text | 1 | |
| P34 | 29–30 Sep | First result window via POST `multi_search`; `$typesense_search_filter_max_ops` guard (100); reference-join clause costed separately | max ops | 3 | |
| P35 | 30 Sep | Partial-index fields: index RS-style fragments as `field_<ref>_p` (chosen over Typesense infix); field:value on such fields filters on `_p` | fragments | 1, 3 | |
| P36 | 30 Sep | Code comments terse, in the file's existing style; measurements and rationale go in the docs | feedback | 0 | |
| P37 | 30 Sep | Inactive fields excluded from positive matching and from the attribute index (staged, not committed) | inactive fix | 1, 6 | |
| P38 | 2 Oct | Review only, no fixes, until discussed; core bugs left as they are | review session | 0, 11 | |
| P39 | 2 Oct | Harness lives in `plugins/typesense_search/harness/` (not `tests/`, which core's runner scans); the branches are not pushed by Claude | harness | 10 | |
| P40 | 2 Oct | Never delete another session's files, worktrees or branches | feedback | 0 | |

Decisions explicitly **still open** when the restart began (now questions in the chunks):

- Validation strictness: drop stale rows and repair, or fall back for the whole search (Q6.1).
- Relevance model: accept, index `hit_count`, or veto keyword-less relevance (Q7.1).
- Type-name and contributor-name keywords: veto or index (Q2.9).
- Collection access gate: core helper (G1) or plugin copy (G2) (Q5.1).
- Which vetoed specials get modes first (Q4.1).
- Date-range semantics: core's string comparison or interval overlap (Q3.4).
- Wildcards over non-indexed fields; NOT over hidden/inactive fields (Q2.3, Q6.10).
- Commit the harness into the plugin (Q10.1).
- Is searching *for* collections in scope (Q5.5).
- Typesense-only mode's purpose (Q0.6).
