# The build plan

The plugin, written again, as small tasks grouped by code area and ordered so each one has what it
needs. The background facts and the full question bank are in [../chunks/](../chunks/); each task
below carries only the facts it must respect and the questions that block it.

## Areas

| Area | File | Builds |
|---|---|---|
| A | [Foundations](A-foundations.md) | Plugin skeleton, Typesense client, schema module, date parsing, text normalisation |
| B | [Indexing](B-indexing.md) | Field mapping, the three document builders, the generation rebuild, schema versioning |
| C | [Sync](C-sync.md) | Queue, request-end flush, worker, hooks, bulk kinds, read-time validation |
| D | [Search pipeline](D-search-pipeline.md) | Hook entry, context/parse, query plan, keyword step, restrictions, sort, execute, hydrate |
| E | [Modes](E-modes.md) | One task per special search, existing and new |
| F | [Field, date, numeric and node forms](F-field-date-numeric-node.md) | Field lookup and every `field:value`, date, number and `@@` form; keyword semantics |
| G | [Operations](G-operations.md) | Setup page, logging, badge, docs, deployment |
| H | [Verification](H-verification.md) | Harness, batteries, live runner, catalogue, accepted divergences |
| I | [Core changes](I-core-changes.md) | The fork's patch, hooks, helper, bug dispositions, upstream tracking |

## Proposed module layout (Q0.14)

```
plugins/typesense_search/
  typesense_search.yaml  config/config.php  languages/en.php
  hooks/all.php                   entry points only; no logic
  include/
    client.php                    A2  requests, multi_search, imports, breaker
    schema.php                    A3  versioned schema, aliases, generations
    dates.php                     A4  date parsing to representations and epochs
    text.php                      A5  normalisation, cleaning, fragments
    fields.php                    B1  field inventory and representation per field
    documents.php                 B2–B4  the three builders
    rebuild.php                   B5–B6  generation build, swap, verify, migrate
    sync.php                      C1–C5  queue, mark-dirty, flush, worker loop
    search/
      context.php parse.php plan.php keywords.php nodes.php sort.php
      execute.php hydrate.php validate.php
      restrictions/ standard.php access.php group_filter.php featured.php
      modes/ standard.php collection.php last.php list.php resource.php
             contributions.php hasdata.php pending.php unsupported.php …
    collections.php               E2  the access gate (helper wrapper or copy)
  scripts/reindex.php  scripts/sync.php
  pages/setup.php
  harness/                        H  (not tests/: core's runner scans that)
  docs/
```

## Task format

Every task has the same five lines so they can be read quickly and compared:

- **Builds** — the file(s) and function(s) it produces.
- **Size** — S (hours) or M (a day). Nothing larger; a larger piece is split.
- **Depends on** — task IDs that must be done first.
- **Must honour** — verified facts the implementation has to respect (the learnings).
- **Blocking questions** — question IDs from the bank, answered before the task starts.
- **Proves it** — the battery or check that closes the task.

## Build order and milestones

| Milestone | Outcome | Tasks |
|---|---|---|
| M0 Foundations | Plugin skeleton loads; client, schema, dates, text modules unit-tested; the core hook patch settled; harness in the repo | A1–A5, I1, H1 |
| M1 An index from scratch | Builders produce complete documents; a generation rebuild on the fixture and on the test system gives counts equal to MySQL | B1–B6, H2 |
| M2 The first served search | A plain keyword search, with every restriction and read-time validation, served with core's results and row shapes | D1–D11, C7, H3 (first batteries) |
| M3 Freshness | Edits, uploads, state and access changes, deletes, grant and collection changes reach the index; outages fall back | C1–C5, H5 |
| M4 Field, date, number and node forms | Catalogue groups B–E identical or declined | F1–F8 |
| M5 Specials and collections | Catalogue group F; the collection matrix | E1–E9, I2 |
| M6 Keyword semantics and relevance | Catalogue group A; the relevance decision implemented | F9, D9 (relevance) |
| M7 Operations, new specials, core hooks | Setup page, docs, deployment; the new modes; the core hooks PR | G1–G6, E10+, C6, I3–I5 |

Inside a milestone the tasks are listed in order; a task's *depends on* line is the rule, the
listing is the suggestion.

## Questions to answer before M0–M3 can start

These gate the first three milestones. Everything else can wait for its milestone.

- **Policy:** Q0.2 goal and measure · Q0.3 decline or approximate · Q0.4 core changes · Q0.5 target
  environments · Q0.6 Typesense-only mode · Q0.8 working practices · Q0.13 keep the pipeline design ·
  Q0.14 the module layout · Q10.11 reuse the harness
- **Schema (M1):** Q1.1 which fields are indexed · Q1.2 representations · Q1.6 extra fields now or
  later · Q1.8 schema evolution · Q1.10 document ids · Q1.11 size budget · Q7.1 relevance (decides
  whether `hit_count` goes in the schema)
- **Search core (M2):** Q4.3 parsing strictness · Q6.1 validation strictness · Q6.2 gate uses that
  bypass the index · Q6.3 the small access fixes · Q7.3 the row limit · Q2.2 typos · Q2.3 prefix
- **Sync (M3):** Q8.1 re-confirm the sync decisions · Q8.4 the interim for hook-less writes · Q8.5
  Phase 0 order · Q8.6 outage behaviour · Q8.7 freshness target
- **Operations basics:** Q9.1 config options · Q9.2 secrets
- **Core:** Q11.1 policy by category · Q11.2 the hook-return patch
- **Testing:** Q10.1 where tests live · Q10.3 acceptance criteria · Q10.4 reference dataset

Thirty-three questions. Answering them in one sitting unblocks roughly the first half of the plan.
