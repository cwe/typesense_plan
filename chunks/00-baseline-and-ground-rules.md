# Chunk 0 — Baseline, scope and ground rules

Status: planning. Depends on: nothing. Everything depends on this.

## Scope

What "start this project again" means in practice: the starting point, the goal and how success is
measured, the constraints every other chunk works within, and the working practices.

## What we know

- The plugin exists and serves most searches: pipeline architecture, four restrictions, nine modes,
  collection and access parity work, sorting fixes, schema work (stemming, separators, cleaning,
  fragments). See [state-of-play.md](../state-of-play.md).
- 48 known differences from core in eight themes (review), ten core bugs, a catalogue of ~300 search
  forms. Combinations of individually-agreeing terms agree in all but one form (2,481 generated cases).
- The index does not stay in sync (chunk 8) and stale rows are not validated (chunk 6). These two
  matter more than any search form.
- The fork carries a core patch (`do_search.php` hook return handling) that the plugin cannot work
  without.
- Chris's working practices so far: Claude stages changes, Chris reviews and commits; terse comments;
  no deleting other sessions' files; branches not pushed by Claude; measurements live in docs, not
  comments.
- Target versions seen so far: Typesense 30.2 (server and local binary), PHP 8.5 locally; the RS
  version of the fork is not recorded here.

## Candidate work items

- Write down the goal, success measure and policies below as a one-page charter in this repo.
- Decide the fate of the three unmerged/uncommitted pieces: the inactive-field fix (staged), the
  harness branch, the catalogue branch.
- Decide which existing docs survive and where (Q0.9).

## Questions to answer before coding

### Q0.1 — Keep the current code as the base, or rewrite from the plan?
The pipeline (modes + restrictions + shared keyword step) has held up through every addition. The
alternative is a rewrite that keeps the schema knowledge and the test harness but starts the query
code again with everything now known. Recommendation: keep the code; the architecture is not the
problem, the unanswered policy questions are.

### Q0.2 — What is the goal, and how is success measured?
Options: (a) serve as many RS searches as possible from Typesense with results identical to core;
(b) faster search where it is safe, core everywhere else; (c) better search than core, accepting
differences. The measure follows: percentage of catalogue rows identical, the live A/B pass rate,
latency against core, or a mix. The original brief was (a).

### Q0.3 — When exact parity is impossible, decline or approximate?
The plugin can decline (veto → core answers, slower) or serve an approximation (faster, different).
Which is the default rule? Are there approximations that are always acceptable (relevance order?)
and ones that never are (anything affecting what a user may see)?

### Q0.4 — Are changes to ResourceSpace core acceptable, and of what kind?
Categories: the existing hook-return patch; new hooks (node layer, grants, collections, field
config); a shared collection access helper; bug fixes; behaviour fixes to core quirks. For each:
fork-only, upstream PR, or neither. (Chunk 11 goes through the candidates.)

### Q0.5 — What are the target environments?
RS version(s) the plugin must run on; PHP version floor; Typesense pinned to 30.x; single Typesense
node only; which servers beyond the test host (sizes, number of instances per Typesense server).

### Q0.6 — What is Typesense-only mode for?
Today it is a testing aid: a declined search returns nothing instead of falling back. If it is ever a
production mode it fails closed for every access-gate use of search (chunk 6). Testing aid only, or a
supported mode with documented consequences?

### Q0.7 — What scale and usage should the design assume?
Resource and membership counts (the test system has ~115k / ~155k), edit rate, concurrent users,
how much of the traffic is search. These drive sync design, the row limit and the rebuild window.

### Q0.8 — Working practices
Branching (keep working on `typesense`, or a branch per chunk merged into it); who commits and
pushes (Chris has done both so far); review before merge; the plugin version number and when it
bumps; how a change reaches the test host.

### Q0.9 — Where do documents live, and which survive?
Eight docs exist in the plugin (some only on unpushed branches). Proposal: this repo holds planning
and decisions; the plugin keeps user- and developer-facing docs (behaviour spec, architecture,
sync design, review); the catalogue and gaps docs move to the harness folder. Which docs merge,
which are retired?

### Q0.10 — In what order are the chunks tackled, and what is the first milestone?
The README proposes 0 → 10 → 11 → 1 → 8 → 6 → 2 → 3 → 4 → 5 → 7 → 9. A candidate first milestone:
"sync works, stale rows are validated, the harness runs in the repo and is green against an
accepted-divergence list". Agree, or a different first milestone (e.g. close the
returns-everything forms first)?

### Q0.11 — Confirm what is out of scope
Searching *for* collections; editable-only searches; faceting (`facet_by=nodes` has no consumer);
multi-language index; anything the API cannot express. Add or remove items.

### Q0.12 — What to do with the three unmerged pieces now
The staged inactive-field fix (commit? it needs a reindex), the harness branch and the catalogue
branch (merge into `typesense`?), before any new work starts.

### Q0.13 — Keep the pipeline design for the rebuild?
Q0.1 is decided: the code is written again. The *design* is a separate choice: one mode + all
restrictions + a shared keyword step, a query plan compiled to parameters, execute, hydrate. It
absorbed every addition (collections, `J`, grants, group filters, `!last`, field forms) without
changes to the orchestrator. Recommendation: keep the design, rewrite the code. Alternative: a
different decomposition (per-search-form handlers, or a single query builder).

### Q0.14 — Approve the module layout
[tasks/README.md](../tasks/README.md) proposes the files: `client`, `schema`, `dates`, `text`,
`fields`, `documents`, `rebuild`, `sync` at `include/`, and `search/` split into context, parse,
plan, keywords, nodes, sort, execute, hydrate, validate with `restrictions/` and `modes/` folders.
Agree, or change the split before tasks are cut to it.

## Done when

A charter page exists in this repo with the answers; the branches/staged work have a decided fate;
chunk statuses reflect the agreed order.
