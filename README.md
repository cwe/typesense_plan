# Typesense search for ResourceSpace — planning

This repository holds the plan for writing the `typesense_search` ResourceSpace plugin **again**: the
code areas to build, each as small ordered tasks; the facts learned so far that each task must
respect; the questions that must be answered before each task starts; and the decisions as they are
made. The code lives in `cwe/resourcespace` (branch `typesense`, `plugins/typesense_search`).
Nothing here is code.

The existing plugin is not the base; what was learned from it is. [principles.md](principles.md)
says how this repo is used to stop fragmented development.

## Start here

1. [principles.md](principles.md) — the ten rules.
2. [state-of-play.md](state-of-play.md) — where the old code, docs, tests and test systems are, and
   what has been measured.
3. [tasks/](tasks/README.md) — **the plan**: the code areas, ~100 tasks, the module layout, the
   milestones, and the 38 questions that gate the first three milestones.
4. [tasks/search/](tasks/search/README.md) — the search side cut to the **core search catalogue**:
   54 packages by catalogue group, each with its rows, fixture and battery, and
   [coverage.md](tasks/search/coverage.md), the row-by-row map of all 300 catalogue rows.
5. [chunks/](chunks/) — the background by subject: verified facts with pointers and the full
   question bank (`Q<chunk>.<n>`). Tasks cite these IDs; answer the questions here.
6. [prior-decisions.md](prior-decisions.md) — 40 decisions taken before the restart, each to confirm
   or revisit.
7. [decisions.md](decisions.md) — the append-only log of answers.
8. [questions.md](questions.md) — generated index of every question.
9. [worksheet.md](worksheet.md) — the gating questions in answering order, with options and
   recommendations, and a line to write each answer on.
10. [catalogue.md](catalogue.md) — the core search catalogue (all 303 cases, what core does, the
   measured core totals) with a **Required** column to mark which cases the rewrite must serve.

## How a task moves

Unblocked (questions answered, dependencies done) → one branch → built with its battery → battery
green and the accepted-divergence list unchanged or deliberately updated → the combinations gate
still green → merged → its rows ticked in [coverage.md](tasks/search/coverage.md) and its status
line updated here. See [principles.md](principles.md).

## Conventions

- No secrets, host addresses or credentials in this repo.
- References to code use the `cwe/resourcespace` path and, where it matters, the branch or commit.
- Findings keep the numbering of the plugin's own docs (review themes A–H, Core 1–10, catalogue rows,
  sync-plan decisions) so the two repos stay in step.
- Questions are decisions for Chris, not research tasks. A recommendation is marked as one.

## The chunks (background and question bank)

| # | Chunk | One line |
|---|---|---|
| 0 | [Baseline, scope and ground rules](chunks/00-baseline-and-ground-rules.md) | Goals, constraints, working practices, where docs live |
| 1 | [Index schema and document model](chunks/01-index-schema-and-document-model.md) | What the index holds and how values are represented, cleaned, stemmed and migrated |
| 2 | [Free-text keyword semantics](chunks/02-free-text-keyword-semantics.md) | AND/OR/NOT, phrases, stop words, prefix, wildcards, typos, synonyms, type-name words |
| 3 | [Field, date, numeric and node searches](chunks/03-field-date-numeric-and-node-searches.md) | `field:value` in all its forms, dates and ranges, numbers, `@@` nodes and their limits |
| 4 | [Special searches](chunks/04-special-searches.md) | Which `!` searches to serve, argument parsing, restypes and workflow states on specials |
| 5 | [Collections](chunks/05-collections.md) | `!collection`, the `J` restriction, memberships, the access gate, hydrated columns |
| 6 | [Access, permissions and read-time validation](chunks/06-access-permissions-and-validation.md) | Grants, group filters, pending rules, searches used as access gates, stale-index validation |
| 7 | [Sorting, relevance and result windows](chunks/07-sorting-relevance-and-result-windows.md) | Sort mapping, the relevance model, `fetchrows` shapes, row limits, next/previous |
| 8 | [Index sync and freshness](chunks/08-index-sync-and-freshness.md) | Hooks, queue, worker, nightly alias rebuild, outage behaviour, the core hooks needed |
| 9 | [Reindexer, configuration and operations](chunks/09-reindexer-configuration-and-operations.md) | Rebuild, performance, config options, setup page, secrets, deployment, monitoring |
| 10 | [Testing and verification](chunks/10-testing-and-verification.md) | The local A/B harness, the live API runner, the catalogue, acceptance criteria, test data |
| 11 | [Core changes and upstream](chunks/11-core-changes-and-upstream.md) | The fork's core patch, hooks and helpers needed in core, core bugs to raise or copy |
