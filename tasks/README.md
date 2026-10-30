# The build plan

The plugin, written again, as small tasks grouped by code area and ordered so each one has what it
needs. The background facts and the full question bank are in [../chunks/](../chunks/); each task
below carries only the facts it must respect and the questions that block it.

The search forms are cut to the **core search catalogue** (groups A–K, 300 rows): one work package
per mechanism, each owning its rows, with its own fixture and battery — see
[search/README.md](search/README.md) and the row-by-row map in [search/coverage.md](search/coverage.md).

## Areas

| Area | File | Builds |
|---|---|---|
| A | [Foundations](A-foundations.md) | Plugin skeleton, Typesense client, schema module, date parsing, text normalisation |
| B | [Indexing](B-indexing.md) | Field inventory, the representation registry (the extension point for every later field), the three builders, import, the generation rebuild, schema versions — mapped by the [schema register](B-schema-register.md) |
| C | [Sync](C-sync.md) | Queue, applying rows, flush, worker, the events through existing and new hooks, bulk and config events, read-time validation — mapped by the [event register](C-event-register.md) |
| D | [Search pipeline](D-search-pipeline.md) | Hook entry, context/parse, query plan, keyword step, restrictions, sort, execute, hydrate — the plumbing every search form needs |
| S | [Search packages](search/README.md) | 54 packages by catalogue group: keywords (SA), field text (SB), fixed lists and nodes (SC), dates (SD), numbers (SE), specials (SF), further sorts (SG), combinations (SH), form strings (SJ), indexing flags (SK) |
| G | [Operations](G-operations.md) | Setup page, logging, badge, docs, deployment |
| H | [Verification](H-verification.md) | Harness, batteries, live runner, catalogue, accepted divergences |
| I | [Core changes](I-core-changes.md) | The fork's patch, hooks, helper, bug dispositions, upstream tracking |
| X | [Security and data protection](X-security.md) | Keys, transport, web exposure and CLI guards, the security battery, erasure and retention, the pre-pilot review, the processing description (X because J and K are catalogue groups) |

(E and F — the earlier "modes" and "field forms" areas — are absorbed into S; the letters are not
reused so that older references stay unambiguous.)

## Module layout (Q0.14, decided 2026-10-16)

One file per concept, each a few hundred lines; the entry point as ResourceSpace developers expect
it; one file per mode and per restriction because that is where parallel work and the "add a
special search" extension happen.

```
plugins/typesense_search/
  typesense_search.yaml  config/config.php  languages/en.php  dbstruct/table_typesense_search_sync_queue.txt (C1)
  hooks/all.php                   includes include/typesense_search_functions.php only; the C5, C6, C9 handlers just mark rows
  include/
    typesense_search_functions.php  the loader and the shared helpers: requires the index and sync modules at load,
                                    the search/ modules only inside the search hook
    client.php                    A2  the client contract: size routing, chunking, breaker, timeouts
    client_curl.php               A2  the curl transport (swappable for the official library later)
    state.php                     A6  generation, ready, bulk_pending, snapshots, counters
    tenancy.php                   A8  the prefix: validation, every server-side name, identity, clash check
    schema.php                    A3  versioned schema, aliases, generations
    dates.php                     A4  date parsing to representations, keys and epochs
    text.php                      A5  normalisation, cleaning, stemming, fragments
    fields.php                    B1  field inventory
    representations.php           B2  the registry: every index field, its value function, version, owner
    documents.php                 B3–B6  resource, membership and grant builders
    import.php                    B7  chunked imports, per-line errors, count verification
    rebuild.php                   B8–B9  generation build, swap, verify, versions, patch
    sync.php                      C1–C4, C7  queue, apply, flush, worker, bulk
    collections.php               SF2  the access gate (helper wrapper or copy)
    search/
      run.php                     D1  the driver: context → parse → build → execute → hydrate
      parse.php                   D2  context and parse
      plan.php                    D3  the plan and the compiler — no HTTP in this file
      escape.php                  D12 every value that enters an expression or SQL
      criteria.php                D4  keywords, field terms, nodes: the shared step
      sort.php                    D9
      execute.php                 D10 the only search file that calls the client; windows, recent-N, merges
      hydrate.php                 D11
      validate.php                C8
      restrictions/ standard.php access.php group_filter.php featured.php
      modes/ standard.php collection.php last.php list.php resource.php contributions.php hasdata.php
             pending.php unsupported.php  (later: properties.php empty.php related.php images.php
             flags.php geo.php colour.php unused.php)
  scripts/reindex.php  scripts/sync.php  scripts/explain.php (G7)   each CLI-guarded; .htaccess deny (X3)
  job_handlers/typesense_search_reindex.php   B8, as the clip plugin does it
  lib/                            A7  (dropped 2026-10-20: no dependency to vendor; the procedure stays for a later swap)
  pages/setup.php
  tests/<n>_typesense_smoke.php   H1 the one-file bridge core's runner scans
  harness/                        H  in the plugin on GitHub; left out of the manual SVN move (Q12.6)
    ab/<ID>_<slug>.php            one battery per task or package, labels = catalogue rows
    live/                         the API runner and the catalogue
  docs/                           incl. data-protection.md (X7)
```

Three rules keep it navigable and testable:

1. **Names carry the module.** `typesense_search_<module>_<verb>()`, so `typesense_search_plan_add_filter()`
   is in `plan.php` without looking.
2. **Split at a seam, never grow.** A file passing about six hundred lines is split along a concept
   boundary; the old plugin's 2,230-line functions file was the largest in the repository.
3. **Require only earlier files.** parse → plan → criteria → sort → execute → hydrate, in that order;
   modes and restrictions require the plan and nothing after it; `execute.php` never requires a
   mode; `plan.php` contains no HTTP. This is what lets a unit battery include only the file under
   test, lets the compiler battery and the explain tool stop before anything is sent, and the harness
   checks it mechanically.
## Task format

Every task has the same lines so they can be read quickly and compared:

- **Builds** — the file(s) and function(s) it produces.
- **Size** — S (hours) or M (a day). Nothing larger; a larger piece is split.
- **Depends on** — task IDs that must be done first.
- **Must honour** — verified facts the implementation has to respect (the learnings).
- **Blocking questions** — question IDs from the bank, answered before the task starts.
- **Proves it** — the battery or check that closes the task.

A search package adds **Rows**, **Core**, **Serve**, **Fixture**, **Battery** and **Open questions**
(see [search/README.md](search/README.md)); the pipeline tasks that own catalogue rows carry a
**Rows** line too. The indexing and sync tasks carry **Covers** (their register rows), **Fixture**
and **Battery** lines in the same way.

## Build order and milestones

**Deadline (Chris, 2026-10-20): finished before the end of 2026, shipping inside ResourceSpace 11.1.** Ten
weeks from the date of the decision. The live work is 74 small and 37 medium tasks (parked and
pending-check packages excluded; the eight pending-check packages became required on 2026-10-30, adding
their tasks), which does not fit in full at one developer's pace. **Scope (Chris, 2026-10-21): nothing is cut —
the whole plan ships in 11.1.** That holds only with packages built in parallel sessions once D1–D12
exist, the smoke battery run before every hand-over, review as the only serial step, and the check
register worked early (H8 and H9 pulled forward to serve it).

| Milestone | Outcome | Tasks |
|---|---|---|
| M0 Foundations | Plugin skeleton loads; client, schema, dates, text, state modules unit-tested; the core hook patch settled; harness in the plugin | A1–A6, A8, X1–X3, I1, H1 (A7 dropped: Q9.10) |
| M1 An index from scratch | Builders produce complete documents; a generation rebuild on the fixture and on the test system gives counts equal to MySQL | B1–B9, G8 (the checklist), G6 (the proof), H2 |
| M2 The first served search | Plain words, node tokens and resolved options, with every restriction, the served sorts, windows, shapes and read-time validation; every other `!` and every declined sort falls back | D1–D12, C1, C8, G7, X4, SA1, SA11, SA12, SC1, SC2, SF8, SG5, SH1 (first pool), H3, H4 (first live run), H7 |
| M3 Freshness | Edits, uploads, state and access changes, deletes, grant and collection changes reach the index; outages fall back | C2–C7, I3, C9, X5, H5 |
| M4 Field, date, number, index-flag forms | Catalogue groups B, C (rest), D, E, K and the form strings identical or declined | SB1–SB6, SC3, SD1–SD5, SE1, SK1–SK3, SJ1, H8 |
| M5 Specials and collections | Catalogue group F for the data already indexed; the collection matrix | SF1–SF7, SF17, I2, X6 (the pre-pilot review) |
| M6 Keyword semantics | Catalogue group A beyond plain words; the relevance decision implemented | H9 (the MySQL mode, first), SA2–SA10, D9 (relevance) |
| M7 Further sorts, new specials, operations, core hooks | Sort keys; the specials that need index fields; setup page, docs, deployment; the remaining core changes | SG1–SG4, SF9–SF16, G1–G5, G6 (the page), G8 (the check), H6, I4, I6, X7, G9 |
Inside a milestone the tasks are listed in order; a task's *depends on* line is the rule, the
listing is the suggestion. SH1 (the combinations gate) and SJ1 (the pages' strings) run again at
the end of every milestone from M2.

## How later packages change the index

Many packages after M1 need a field the index does not have yet (the date components, the
option-name array, the date key, the `!properties`, `!related`, image, flag, geo and colour
fields, the sort keys, type names). They never reopen the builder, the rebuild or the worker:
[B-indexing.md](B-indexing.md) states the contract. In short — B2's registry is the only place a
field is added; the package follows the six-step schema-change protocol (entry, register row,
version bump, battery cases, builder-path equality, rebuild on the fixture); a representation read
from a table other than `resource` / `resource_node` / `resource_data` adds its event to
[C-event-register.md](C-event-register.md); deployment is a generation rebuild. The two registers
show, for every field and every event, which package owns it and which version carried it.

## Questions to answer before M0–M3 can start

[../worksheet.md](../worksheet.md) lists these in answering order with their options and a line for
the answer. These gate the first three milestones. Everything else can wait for its milestone — but the five
marked ★ shape several packages at once and are worth answering first.

- **Policy:** Q0.2 goal and measure ★ · Q0.3 decline or approximate ★ · Q0.4 core changes · Q0.5
  target environments · Q0.6 Typesense-only mode · Q0.8 working practices · Q0.13 keep the
  pipeline design · Q0.14 the module layout · Q10.11 reuse the harness
- **Schema (M1):** Q1.1 which fields are indexed ★ · Q1.2 representations · Q1.6 extra fields now
  or later · Q1.8 schema evolution · Q1.10 document ids · Q1.11 size budget · Q7.1 relevance
  (decides whether `hit_count` goes in the schema) · Q2.14 the stemmer ★ (decides the text schema)
- **Search core (M2):** Q4.3 parsing strictness · Q6.1 validation strictness · Q6.2 gate uses that
  bypass the index · Q6.3 the small access fixes · Q7.3 the row limit · Q2.2 typos · Q2.3 prefix ·
  Q3.1/Q3.12 unresolved option text ★ (fixes the everything-class before M4)
- **Sync (M3):** Q8.1 re-confirm the sync decisions · Q8.4 the interim for hook-less writes · Q8.5
  Phase 0 order · Q8.6 outage behaviour · Q8.7 freshness target
- **Operations basics:** Q9.1 config options · Q9.2 secrets · Q9.10 the official client or a custom
  one · Q9.11 where plugin state lives · Q9.12 the prefix policy
- **Core:** Q11.1 policy by category · Q11.2 the hook-return patch
- **Testing:** Q10.1 where tests live · Q10.3 acceptance criteria · Q10.4 reference dataset ·
  Q10.12 battery naming · Q10.13 fixture additions · Q10.15 the combinations gate
- **Security:** Q12.1 keys · Q12.2 network and transport · Q12.3 the metadata copy · Q12.4 the
  stale window and the gate uses · Q12.6 web exposure of plugin files · Q12.11 a server-wide key
  or tenant keys only

Forty-seven questions. The packages' own *open questions* (Q1.12–Q1.13, Q2.15–Q2.18, Q3.13–Q3.18,
Q4.11, Q5.11, Q7.9, Q10.14) are answered when their package starts.
