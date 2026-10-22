# Questions to work through

The questions that gate the plan, in the order worth answering them, each with its options, the
recommendation where there is one, and what it shapes. Fill in the **Answer** and **Reason** lines;
each answer then becomes a row in [decisions.md](decisions.md) (date, ID, decision, reason) and the
chunk file's question is marked decided. The full question texts are in [chunks/](chunks/) and the
index in [questions.md](questions.md).

Rec. = the recommendation already written in the plan. "Shapes" names the tasks and packages whose
*must honour* or *serve* lines change with the answer.
Standing (added 2026-10-20) = what the catalogue marks say about the question: see the end of the file.

## Part 1 — The six that shape the most packages

### Q0.2 — What is the goal, and how is success measured?
(a) as many RS searches as possible served from Typesense with results identical to core; (b)
faster search where it is safe, core everywhere else; (c) better search than core, accepting
differences. The measure follows: % of catalogue rows identical, the live A/B pass rate, latency vs
core, or a mix. The original brief was (a).
Shapes: every `?` row in coverage.md; the accepted-divergence list.
Standing (2026-10-20): **uncited** — no row and no blocking line names it; it acts through its *Shapes* line
- **Answer:**
- **Reason:**

### Q0.3 — When exact parity is impossible: decline or approximate?
Decline (veto → core answers, slower) or serve an approximation (faster, different). Are there
approximations that are always acceptable (relevance order?) and some that never are (anything
affecting what a user may see)? Principle 7 already rules out "everything" and "nothing".
Shapes: SA3, SA5, SA7, SA8, SD3, SG*, D9.
Standing (2026-10-20): **uncited** — no row and no blocking line names it; it acts through its *Shapes* line
- **Answer:**
- **Reason:**

### Q1.1 — Index only flagged fields, or every field's values?
(a) flagged text fields only, plus date and numeric representations for every field (SD5, Q3.16);
(b) every active field's values, with non-indexed fields kept out of `query_by` and word filters and
used only for wildcards (full parity for K2/K3; more memory, Q1.11); (c) flagged only, nothing else.
Rec.: (a) — parity by construction for words, options and field terms; bare wildcards over
non-indexed fields become an accepted difference or a veto (Q2.3).
Note (Q10.3): an approximate set is out, so (a) means those forms decline; (b) means they serve.
Shapes: B1, B2, B4, SA5, SB3, SK1, SD5, SE1.
Standing (2026-10-20): **required** — must rows A11, A27; check rows K2, K3; blocks A3, B1, SA5, SA8, SC3, SK1
- **Answer:** (b), for now — every active field's values are indexed; the flags decide reach, not presence. Plus an exclusion list on the setup page (G9). Revisit against different data sets (H8).
- **Reason:** measured 2026-10-20: the non-indexed text is one field on each test database (26.7 MB of 37 MB on the first, 16.5 MB against 494 MB on the second), so (b) is cheap and attributable, while (a) would decline every wildcard and full-text search on both. See decisions.md.

### Q2.14 — Pre-stem with ResourceSpace's own stemmer on both sides?
Core stems with `GetStem()` (irregular plurals, then Porter). Typesense's Snowball agrees on 34 of
40 word forms, and the mismatch is why every quoted phrase returned 0 under `$stemming`. Options:
pre-stem values and query words with `GetStem()` and turn Typesense stemming off (identical stems by
construction; quoted words sent unstemmed as core does; phrases need no special rule — needs a
check that the tokeniser leaves pre-stemmed tokens alone); keep Typesense's stemmer and list the
differing forms, solving phrases under Q2.5; or trial both on the fixture first.
Rec.: pre-stem.
Note (Q10.3): drift changes sets, so the choice is pre-stem or decline under `$stemming`.
Shapes: A3, A5, B2, SA3, SA12, Q2.5.
Standing (2026-10-20): **required** — must rows A5; blocks B2
- **Answer:**
- **Reason:**

### Q3.1 / Q3.12 — Option text core does not resolve
The old plugin returned everything for `usagerights:social`, `artistname:moore`, `soc*` and a
BOM-prefixed name. Q3.1 says decline; Q3.12 proposes serving them as a word match on the field's
option names (`_ss`, tokenised and stemmed like text): parity for indexed fixed-list fields, and a
non-indexed fixed-list field has no `_ss` and matches nothing, as core. Variant: serve the word
match, veto wildcards and BOM names until proven.
Rec.: serve via `_ss`.
Shapes: SC3, B2.
Standing (2026-10-20): Q3.1: **required** — blocks SC2, SC3 · Q3.12: **required** — must rows B9, C3, C9; check rows C4, C5, C7, C12; blocks B2, SC3
- **Answer:**
- **Reason:**

### Q3.4 — Date-range semantics
(a) replicate core's string comparison with a fixed-width sortable key (padded ends `2024-02-31`
and EDTF `2024-12-99` give core's answer; year-only stored values never match a range starting
inside their year, as in core); (b) interval overlap on `_range_start`/`_range_end` (more correct,
+106 on the test system), documented; (c) overlap with non-real ends clamped.
Rec.: (a) if Q0.2 is parity — the advanced form produces these ends routinely.
Note (Q10.3): overlap as a served difference is out; (a), or decline ranges with non-real ends.
Shapes: A4, SD3, SD4, H35, J5, J6.
Standing (2026-10-20): **required** — must rows D6, D19, H35, J5, J6; check rows D5, D7, D8, D9; blocks A4, B2, SD3
- **Answer:** (a) a fourteen-digit integer key `_k` reproducing core's string comparison, non-real ends included; `_k` enters version 1, the range pair leaves it.
- **Reason:** exact on every row; a stored year-month is outside a day or month range in core too; overlap would be the first set difference on the accepted list. decisions.md 2026-10-21.

## Part 2 — The rest of the gating questions (M0–M3)

### Policy

#### Q0.4 — Are changes to ResourceSpace core acceptable, and of what kind?
Categories: the existing hook-return patch; new hooks; a shared collection access helper; bug
fixes; behaviour fixes to quirks. For each: fork-only, upstream PR, or neither (Q11.1 asks it per
category).
Shapes: I1–I5, SF2, C9, D10 (next/previous).
Standing (2026-10-20): **required** — blocks I1
- **Answer:** yes, any kind: core changes are made in ResourceSpace itself (see Q11.1).
- **Reason:** Chris works for Montala; a change the plugin needs is a core change in a release, and the plugin states the minimum version it needs.

#### Q0.5 — Target environments
RS version(s); PHP floor; Typesense pinned to 30.x; single node; which servers beyond the test host.
Shapes: A1, A2, G5.
Standing (2026-10-20): **required** — blocks G8, X2
- **Answer:** (in part) ResourceSpace 11.1 — the plugin ships inside it (Q11.6), so the PHP floor is 11.1's; Typesense 30.x; the servers beyond the test host still to state.
- **Reason:** Chris, 2026-10-20: the plugin will ship with ResourceSpace 11.1 as part of ResourceSpace.

#### Q0.6 — What is Typesense-only mode for?
A testing aid (a declined search returns nothing instead of falling back) or a supported production
mode? As a production mode it fails closed for every access-gate use of search (Q6.2, Q6.5).
Shapes: D1, C8, G1.
Standing (2026-10-20): **required** — blocks D1
- **Answer:** a testing aid only — group override for test groups, never real users; returns nothing in an outage; the setup page says so.
- **Reason:** as a production mode it empties every declined form and fails closed on the access gates. decisions.md 2026-10-20.

#### Q0.8 — Working practices
Branching (keep `typesense`, or a branch per package merged into it); who commits and pushes (Chris
so far); review before merge; plugin version bumps; how a change reaches the test host.
Shapes: principles 3 and 10, G5.
Standing (2026-10-20): **uncited** — no row and no blocking line names it; it acts through its *Shapes* line
- **Answer:**
- **Reason:**

#### Q0.13 — Keep the pipeline design for the rebuild?
The design (one mode + all restrictions + a shared keyword step, a query plan compiled to
parameters, execute, hydrate) absorbed every addition without orchestrator changes. Rec.: keep the
design, rewrite the code. Alternative: per-form handlers or a single query builder.
Shapes: D1–D11, every S package.
Standing (2026-10-20): **required** — blocks A1
- **Answer:** (a), written procedurally; the plan admits AND clauses plus five declared mechanisms
  (recent-N, suppression, the `J`-in-collection rule, OR alternatives, budgets) and nothing else;
  SH1 gates every merge. Decided 2026-10-16.
- **Reason:** core ANDs everything, so combinations are the sum of their parts; the five mechanisms
  are the exceptions, four already proven in the old code. See decisions.md.

#### Q0.14 — Approve the module layout
The file split in [tasks/README.md](tasks/README.md). Agree, or change the split before tasks are
cut to it.
Shapes: every task's *builds* line.
Standing (2026-10-20): **required** — blocks A1
- **Answer:** option 3 — one file per concept, one per mode and restriction, the loader at the
  conventional entry point, with the naming, split-at-a-seam and require-only-earlier-files rules.
  Decided 2026-10-16.
- **Reason:** the most maintainable split: readable files, small diffs, parallel work on separate
  files, isolated unit batteries. See decisions.md.

#### Q10.11 — Reuse the existing harness code?
The harness (SQLite for MySQL, a private Typesense, real core and plugin run twice per case, the
fixture library, the batteries, the factorial combinations) is test infrastructure, not plugin code.
Rec.: reuse it reviewed (principle 6), as task H1, re-cutting only the batteries that assumed old
behaviour.
Shapes: H1, every battery.
Standing (2026-10-20): **required** — blocks H1
- **Answer:** option (c): reuse reviewed now (H1: three couplings rewritten, batteries re-cut per
  package, trace harness retired, results cleared) and a MySQL-backed mode later (H9) for the forms
  the shim emulates. Decided 2026-10-16.
- **Reason:** the shim is the proven hard part; its emulations matter more now that wildcards and
  the full-text box are must, and a second driver removes that caveat cheaply. See decisions.md.

### Schema (M1)

#### Q1.2 — Which representations per field type?
Drop `_q` for text fields (unused); add sortable keys for the sorts chosen in Q7.2; keep `_ts` when
`_range_start` covers every case; add `_k` (string-comparison key) if Q3.4 chooses (a); `_ss` for
fixed-list fields (Q3.12); integer date components for `$date_field` (Q3.13).
Shapes: A3, B2, SD2, SD3, SG1–SG4.
Standing (2026-10-20): **required** — blocks A3, B2
- **Answer:** (a) version 1 = the old set minus `_q` on text, `_ts` and the facet flag on `_s`; nothing speculative. A representation enters only with the package that proves it; additions batched into one rebuild per release.
- **Reason:** a later addition is one schema version and one generation rebuild, which the nightly job does anyway; speculative fields cost memory and proof time M0 does not have. decisions.md 2026-10-20.

#### Q1.6 — Which extra fields go in now?
`hit_count`/`user_rating` (relevance, popularity, rating sorts); `has_image`; file size, extension,
width, height; `lock_user`, `integrity_fail`, file presence; a geopoint; `colour_key`; related refs;
`rty.order_by`; a sortable title. One batch (one rebuild) or per package (Q4.2)?
Shapes: A3, B2, SF9–SF16, SG1–SG3.
Standing (2026-10-20): **required** — blocks A3
- **Answer:** (b) base rows plus the hit-count cluster (`hit_count`, `user_rating`, `rating`, the packed keys), conditional on Q7.1 and the G11 check; everything else with its package, batched per release.
- **Reason:** the cluster is the one extra an M2 task needs — keyword-less relevance order, the default order of the search page; the rest belongs to 11.2 packages. decisions.md 2026-10-20.

#### Q1.8 — Schema evolution
Generation collections + aliases (P7); what triggers a full rebuild (field config, `$stemming`,
schema version); how the version is recorded; whether `PATCH` stays for additive changes.
Shapes: B8, B9, G5.
Standing (2026-10-20): **required** — blocks A3, B9
- **Answer:** rebuild only — every collection-definition change is a generation rebuild, no PATCH; version in the generation metadata; "rebuild required" served from the old generation. Schema changes only: the document-level mechanisms are the sync questions' (Q8.x), later.
- **Reason:** PATCH alters the serving collection, was slow, and can leave schema and builder out of step; additions are batched per release anyway (Q1.2). decisions.md 2026-10-20.

#### Q1.10 — Document identity
Resources `id = ref`; memberships `collection:resource`; grants: confirm the scheme.
Shapes: B3–B6.
Standing (2026-10-20): **required** — blocks A3, B6
- **Answer:** (a) natural keys: resources `id` = ref as a string (plus integer `ref`, `ref_s`); memberships `collection:resource`; grants `resource:user:usergroup` with `-1` unused; one id function per collection in B2.
- **Reason:** idempotent imports make queue replay and the rebuild catch-up safe; deletes address by id; duplicate rows collapse. decisions.md 2026-10-20.

#### Q1.11 — Size budget
~800 MB today for ~115k resources with fragments; two generations briefly during a rebuild. What is
acceptable on the target, and does it rule anything out (Q1.1, Q1.6, Q1.3)?
Shapes: B8, SK1, SB6.
Standing (2026-10-20): **required** — blocks A3, B8
- **Answer:** no budget rule: room to serve one generation and build the next is the requirement; B8 checks memory before a build and refuses when the second generation would not fit; G8 states sizing guidance from the measured points; levers documented, nothing cut in advance.
- **Reason:** memory is a server-sizing fact, not a design constraint. decisions.md 2026-10-20.

#### Q7.1 — The relevance model
(A) accept Typesense's text match for keyword searches and newest-first for keyword-less ones,
documented; (B) index `hit_count` and `user_rating` and sort keyword-less searches by `hit_count,
date_field_sort, ref` (387 of core's top 400; fresh only after a rebuild) — keyword searches still
differ; (C) veto keyword-less relevance meanwhile. And: is a different relevance order acceptable to
the users at all?
Note (Q10.3): keyword-less relevance must be exact — a composite sort key, exact as of the last
rebuild; the remaining choice is whether that staleness is acceptable.
Shapes: D9, A3, SG2, SC1 (node-only order), SF2 (Q5.3).
Standing (2026-10-20): **required** — check rows G11; blocks A3, B3, D9, SG2 (pending check)
- **Answer:** (B) the hit-count cluster in version 1; keyword-less relevance exact via the packed keys, kept fresh by a new hook in `copy_hitcount_to_live()` plus the rating and edit hooks. Keyword searches ordered by relevance differ from core (per-search node-hit score): the one accepted difference, documented.
- **Reason:** core refreshes hit counts at most daily, so the hook makes the order exact rather than stale; keyword-less relevance is the default order of every browse view. decisions.md 2026-10-20.

### Search core (M2)

#### Q4.3 — Argument parsing: replicate core exactly, or parse strictly and veto?
Core's parsing is inconsistent (space vs comma, every digit in the string, case-sensitive commands,
unknown special → everything). (a) replicate, bugs included; (b) parse each special's documented
form and veto the rest (core then answers — the user sees core's behaviour either way); (c) parse
sensibly and accept differences. Rec.: (b).
Shapes: D2, SF1, SF3, SF4, SF8.
Standing (2026-10-20): **required** — must rows F3; bug rows F4, F31; blocks D2, SF1, SF3, SF4
- **Answer:**
- **Reason:**

#### Q6.1 — Validation strictness
(A) drop stale rows, keep the total, queue a repair; (B) fall back to MySQL for the whole search when
any row is dropped, queue a repair, count it. Rec.: (B) — a dropped row means the total and paging
are wrong too. Mechanism: re-run `search_filter()` plus the `rca`/`rca2` joins in the hydrate query.
Shapes: C8, D11.
Standing (2026-10-20): **required** — blocks C8, D11
- **Answer:**
- **Reason:**

#### Q6.2 — Gate uses that must not depend on the index
`get_resource_access()`'s `!resource<ref>`, the share gate, request copies, submit-pending: detect and
veto (single-resource, refs-only or count-only searches), rely on validation and accept the window,
or a core change so those call sites bypass external providers.
Shapes: SF4, SF5, C8, D1.
Standing (2026-10-20): **required** — blocks C8, X4, SF4
- **Answer:** a core change: the gate call sites bypass the `external_search` hook (decided with Q12.4).
- **Reason:** `get_resource_access()` runs `r.ref = N` plus the filter conditions — milliseconds in MySQL; validation alone would still wrongly deny for up to a minute; shape detection is a heuristic. decisions.md 2026-10-22.

#### Q6.3 — The small fixes, as a batch
`ignore_filters` no longer disables restrictions; grant expiry honoured in the group filter's
override; group filter applied under `access_override`; `J` with no `j` → nothing; hook parameters
nullable where core passes null. Approve?
Shapes: D1, D5, D7, D8.
Standing (2026-10-20): **required** — blocks D1, D5, D7, D8, D12
- **Answer:**
- **Reason:**

#### Q7.3 — The row limit and next/previous
Keep 25,000? For `view.php`'s next/previous (`fetchrows = -1`): serve fully (20–37 s on the whole
index), force navigation to core (order differs from the grid), or change core's navigation to a
windowed query (Q0.4). Rec.: the core change.
Shapes: D10, I4.
Standing (2026-10-20): **required** — blocks D10, I4
- **Answer:**
- **Reason:**

#### Q2.2 — Typo tolerance
Core has none. `num_typos=0` for parity, or allow typos as a deliberate improvement?
Shapes: D3, SA1.
Standing (2026-10-20): **required** — blocks D3, SA1
- **Answer:**
- **Reason:**

#### Q2.3 — Prefix and wildcards
`prefix=false` by default (fixes `land`). For `word*`: raise `max_candidates` (to what?) and
`max_filter_by_candidates` for `field:word*`. Leading and middle wildcards: veto, or serve from
fragments where a field has them. Wildcards over non-indexed fields: core finds them — veto bare
wildcards, index everything (Q1.1), or accept.
Shapes: D3, SA1, SA5, SB3, SK1.
Standing (2026-10-20): **required** — must rows A11; check rows A12, B7, H22, K2; blocks D3, D4, SA1, SA5, SB3, SK1
- **Answer:**
- **Reason:**

### Sync (M3)

#### Q8.1 — Re-confirm the thirteen sync decisions (P7–P17, P24)
Nightly rebuild behind aliases (often enough? a window available?); a queue table drained by a
per-minute cron worker (system cron available, or only RS's `hook("cron")`?); MySQL serves while
bulk rows are pending; the plugin global with per-group enable; no deferred bulk work.
Shapes: B8, C1–C7, A1.
Standing (2026-10-20): **required** — blocks B8, C1, C2, C4, C7, G8
- **Answer:**
- **Reason:**

#### Q8.4 — Core hooks (Phase 2) and the interim
Confirm the hook list (node layer, resource type, custom access, `put_resource_data`, collection
bulk writes, field-config arguments) and that hook-less writes wait for the nightly rebuild until
then.
Shapes: I3, C5, C6, C9.
Standing (2026-10-20): **required** — blocks C5, C6, C9, I3
- **Answer:**
- **Reason:**

#### Q8.5 — Phase 0 scope and order
Builder → queue + flush + worker → hooks → guards → validation in Phase 0 → generation rebuild.
Anything to pull forward (the generation rebuild first, since it fixes freshness nightly on its own)?
Shapes: the order of B8 and C2–C7.
Standing (2026-10-20): **required** — blocks C1
- **Answer:**
- **Reason:**

#### Q8.6 — Outage behaviour
Circuit breaker + fall back to MySQL (never empty results); the queue keeps accumulating; the worker
retries with backoff. Confirm, and how Typesense-only mode behaves in an outage (Q0.6).
Shapes: A2, C4, D1.
Standing (2026-10-20): **required** — blocks A2, B7, C4
- **Answer:** confirmed — breaker on transport failures and 5xx only, open = immediate fallback, half-open probe on `/health`, queue keeps marking, worker backs off and drains, rebuild runs when the server is back, breaker state on the setup page. Never an empty result for an outage.
- **Reason:** the design as written; the only open part was Typesense-only mode (Q0.6). decisions.md 2026-10-20.

#### Q8.7 — Freshness target
Own edits on the next page load; others' within seconds; bulk changes intraday with MySQL serving
meanwhile; hook-less writes nightly. Any flow needing synchronous visibility?
Shapes: C3, C4, C7.
Standing (2026-10-20): **required** — blocks C3, C7
- **Answer:**
- **Reason:**

### Operations basics

#### Q9.1 — Configuration
The option list and defaults (prefix, timeouts, max rows, max filter ops, import concurrency and
chunks, worker count, validation, Typesense-only, rebuild schedule display); which are admin-facing.
Shapes: A1, A2, G1.
Standing (2026-10-20): **required** — blocks A1, A2, G1
- **Answer:** (a) with a lock rule: any page option fixable from `config.php` via `_override`, drawn locked ("locked in global configuration"), keys hidden when locked. Page: server address, two keys (password fields), enabled, badge, excluded fields; health read-only; rebuild / resync lockable; tuning and max rows `config.php` only; Typesense-only off the page; validation not optional; global filter dropped.
- **Reason:** the openai_gpt pattern Montala's hosting uses to keep keys outside client control; stored options overwrite same-named `config.php` values, so overrides are separately named. decisions.md 2026-10-21.

#### Q9.2 — Secrets
The API key as plugin config in the database (set from the setup page) — acceptable? Separate
search-only and admin keys?
Shapes: A1, A2, G1.
Standing (2026-10-20): **required** — blocks A1
- **Answer:**
- **Reason:**

#### Q9.10 — The official Typesense PHP client, or a custom client?
(a) the official `typesense-php` client vendored in the plugin's `lib/` behind a thin adapter that
keeps the size routing, chunking, breaker and timeouts (rec.); (b) the custom curl client as A2 is
written; (c) the official client via the fork's root `composer.json`. The library covers every
endpoint with typed exceptions but has no GET/POST choice, no concurrency and no breaker.
Shapes: A2, B7, C4.
Standing (2026-10-20): **required** — blocks A2, A7, B7
- **Answer:** (b) the custom curl client, behind a transport seam (`client.php` contract + `client_curl.php` transport) so the official library can be swapped in later; A7 dropped from M0, kept as the swap procedure.
- **Reason:** inside ResourceSpace 11.1 the library means a vendored stack in the core repository for little saving; the curl client is proven and can import in parallel. decisions.md 2026-10-20.

#### Q9.11 — Where plugin state lives
Core's `sysvars` table under a prefixed key (rec.: no new table, visible to core's tooling) or a
plugin table with typed columns; MySQL either way, not ready by default on a fresh install.
Shapes: A6, B8, B9, C4, C7, G1.
Standing (2026-10-20): **required** — blocks A6
- **Answer:** (b) a plugin table `typesense_search_state` (prefix, name, value, updated) via dbstruct; atomic set, in-place counters; entries listed in decisions.md.
- **Reason:** `sysvars` writes are delete-then-insert, null-equal values delete the row, and its two caches serve stale reads to long-running processes; the three per-search flags must never be stale. decisions.md 2026-10-21.

#### Q9.12 — The prefix policy for tenants
Mandatory and validated with the installation's identity written into each generation and a clash
check (rec.), or derived from `$baseurl` with an override. Q9.8 is decided: several installations on
one server is a requirement.
Shapes: A8, A1, B8, G6.
Standing (2026-10-20): **required** — blocks A8
- **Answer:** (b) a random prefix generated at first setup, stored, read-only on the page, overridable from `config.php`; identity clash check kept; lockable "regenerate prefix" for clones.
- **Reason:** no clashes by construction, nothing for a client to break. decisions.md 2026-10-21.

### Core

#### Q11.1 — The policy, by category
Fork-only, upstream PR, or neither — for the hook-return patch, new hooks, the collection gate
helper, the next/previous and page-offset fixes, bug fixes, quirk fixes.
Shapes: I1–I5.
Standing (2026-10-20): **required** — blocks C9, I1, I2, I3, I6
- **Answer:** every category is a core change in ResourceSpace: the hook reorder, the sync hooks, the collection gate function, next/previous and the page offset when wanted; bugs and quirks fixed in core by default, Q11.3 per row.
- **Reason:** Chris works for Montala, so the fork-only / upstream split does not exist. decisions.md 2026-10-20.

#### Q11.2 — The existing hook-return patch
A prerequisite that must go upstream for the plugin to work on stock RS? Was it commented out
upstream for a reason worth knowing before proposing it?
Shapes: I1.
Standing (2026-10-20): **required** — blocks I1
- **Answer:** the reorder as it is in the fork, into core, no gating or second hook.
- **Reason:** upstream never had the return handling commented out; the only change is the order, and it is needed for every special search. Chris: no fancy workaround.

### Testing

#### Q10.1 — Where the tests live
`harness/` in the plugin on `typesense` (not `tests/`), merging the two unpushed branches; or a
separate test repository. Rec.: in the plugin.
Shapes: H1.
Standing (2026-10-20): **required** — blocks H1
- **Answer:** option (a): `plugins/typesense_search/harness/` on `typesense`, guarded and excluded
  from deployments, results not committed; the two branches merged by Chris first. Decided
  2026-10-16.
- **Reason:** batteries move with the code; one checkout for CI and upgrades; principle 3 holds. See
  decisions.md. Amended 2026-10-22 (Q12.6): location moved to the planning repository.

#### Q10.3 — Acceptance criteria
"Pass" = identical totals and order for every catalogue row core answers sensibly, with an
accepted-divergence list the runners consult. Who maintains the list; does an unlisted difference
fail the run?
Shapes: H1, H3, SH1, coverage.md.
Standing (2026-10-20): **required** — blocks H1, SH1 (pending check)
- **Answer:** served = exact set, total and order; relevance-to-text order the only listed
  exception; declines match their planned reason; core bugs per Q11.3; unlisted differences fail;
  the list in the harness, every entry citing a decision, Chris approving at merge. Decided
  2026-10-16.
- **Reason:** "anything we want Typesense to support that doesn't involve relevancy to a text string
  should exact match". It directs Q3.4, Q2.14, Q1.1, Q1.13, Q6.7 and Q7.1 — see decisions.md.

#### Q10.4 — Test data
The YSP database (real, large, limited field types) or a seeded system (chosen data: a tree, dynamic
keywords, a date-range field, numeric fields, HTML and translated values, related keywords, search
filters, custom grants, `J`/`j*` users)? Extend the seeder for the missing kinds?
Shapes: H4, every live battery.
Standing (2026-10-20): **required** — blocks H4, H8
- **Answer:**
- **Reason:**

#### Q10.12 — Battery naming tied to the catalogue
One harness file per package, case labels carrying catalogue row IDs, `summarise.php` filling the
catalogue's OK column, `catalogue_run.php <IDs>` for a package's live rows. Agree?
Shapes: H1, H3, every package's *battery* line.
Standing (2026-10-20): **required** — blocks H1
- **Answer:** agreed — one file per package, catalogue row ids as labels, `summarise.php` fills OK and Done, `catalogue_run.php <IDs>` for live rows; old batteries re-cut per package.
- **Reason:** one index of what is proved; the old labels clash with the catalogue's ids. decisions.md 2026-10-21.

#### Q10.13 — Fixture additions per package
Per-package additions registered in the fixture library and loaded only by the batteries that need
them (rec.), or one shared fixture that grows.
Shapes: H1, every package's *fixture* line.
Standing (2026-10-20): **required** — blocks H1
- **Answer:** agreed — per-package additions in the fixture library, loaded only by the batteries that need them; SH1 loads the union.
- **Reason:** packages prove alone, which parallel sessions need. decisions.md 2026-10-21.

#### Q10.15 — The combinations gate per package or per milestone
SH1 (~2.5 minutes at full scale) before every package merge, or once per milestone with a scaled-down
run per package.
Shapes: SH1, principle 3.
Standing (2026-10-20): **uncited** — no row and no blocking line names it; it acts through its *Shapes* line
- **Answer:**
- **Reason:**

### Security

#### Q12.1 — Keys: which, where, shown how
Two keys (a search-only key scoped to `documents:search` on `<prefix>_*`; an admin key for the CLI)
or one; in `config.php` as core keeps its secrets (rec.) or in the plugin config in the database;
the setup page shows set / not set only; a rotation procedure. Supersedes Q9.2.
Shapes: A2, X1, G1, G8.
Standing (2026-10-20): **required** — blocks A2, G1, G8, X1
- **Answer:** two scoped keys per installation (search-only for the request path; write for the rebuild and worker), stored as plugin options, lockable and hidden from `config.php`; set / not set on the page; a provisioning action takes the bootstrap key once and never stores it; rotation = provision again.
- **Reason:** a single key puts write access in the web tier; `config.php`-only shuts out administrators without file access, for whom the lock is the hosted answer. decisions.md 2026-10-22.

#### Q12.2 — Network placement and transport
Typesense on localhost or a private network; TLS mandatory when not local; certificate verification
always on; the endpoints listed for a firewall rule.
Shapes: A2, X2, G8.
Standing (2026-10-20): **required** — blocks A2, G8, X2
- **Answer:** localhost in the clear, TLS everywhere else, certificate verification that cannot be switched off; host/port/protocol validated and lockable; keys only in headers; server side (bind, firewall, CORS off, private CA) and the endpoint list in G8.
- **Reason:** a shared hosted server is never reached in the clear; no installation can be left insecure by a switch. decisions.md 2026-10-22.

#### Q12.3 — Is a full metadata copy in Typesense acceptable?
Accept and protect the server (rec.), exclude confidential resources or hidden fields (granted users
then lose Typesense), or add encryption at rest.
Shapes: B2, B4, X7.
Standing (2026-10-20): **required** — blocks X7
- **Answer:** accept with protection: scoped keys and TLS (Q12.1, Q12.2); X7 states what the index holds and who can read it; G9 and encryption at rest named as the levers.
- **Reason:** excluding confidential or hidden data would make granted users' searches decline wholesale; encryption at rest protects a disk, not a key. decisions.md 2026-10-22.

#### Q12.4 — The stale-row window and the gate uses, as a security requirement
C8 option B in M2; the gate uses bypass the index or are always validated; Typesense-only never for
real users; the acceptable listing window before the hooks land.
Shapes: C8, D1, SF4, X4.
Standing (2026-10-20): **required** — blocks C8, X4
- **Answer:** gate uses bypass external providers through a core change; C8 option B in M2; Typesense-only never for real users; window = request end for own changes, within a minute for others; the newly-permitted-but-absent residue documented in X7.
- **Reason:** an access decision must never depend on index freshness, and the gate search is a primary-key lookup that gains nothing from Typesense. decisions.md 2026-10-22.

#### Q12.6 — Web exposure of plugin files
`.htaccess` deny rules and CLI guards for `scripts/`, `lib/`, `harness/`, the harness excluded
from deployments — or the harness kept outside the plugin (Q10.1).
Shapes: X3, A7, H1.
Standing (2026-10-20): **required** — blocks X3
- **Answer:** (b) provisional — the harness lives in the planning repository, found by a path setting; only the one-file `tests/` bridge ships; `scripts/` gets a `PHP_SAPI` guard and a deny `.htaccess`, page actions a permission check and a form token; no `lib/`.
- **Reason:** ResourceSpace has no release-exclusion mechanism, so separate means another repository; the planning repository holds the coverage table the harness fills. Chris may change this. decisions.md 2026-10-22.

#### Q12.11 — A server-wide key for diagnostics, or tenant keys only?
Tenant keys only (the server check degrades to health and the tenant's own collections), one
read-only server-wide diagnostics key entered per tenant, or server checks done outside RS. Verify
first whether a prefix-scoped key also confines alias and synonym-set calls.
Shapes: X1, G6, G8.
Standing (2026-10-20): **required** — blocks G6, X1
- **Answer:** (b) tenant keys only, but the write key also carries `debug:list` and `metrics.json:list` — read-only server status, no data — so the version check and B8's memory check run with the tenant's own key.
- **Reason:** no key can reach another tenant's data; the two actions expose load and version, verified in X1 to return no collection content. decisions.md 2026-10-22.

## Part 3 — Answered when the package starts

One line each; the full text is in the chunk file. (Q0.1 is decided; Q8.2 is Q6.1 asked twice; Q8.3
is a test to run, not a question.)

- **Q0.7** scale and usage to assume (C, B8) · **Q0.9** where documents live, which survive (G4) ·
  **Q0.10** order and first milestone — the milestones in tasks/README.md are the proposal · **Q0.11**
  confirm what is out of scope · **Q0.12** the three unmerged pieces (H1)
- **Q1.3** fragments free-text searchable? (SB6, K5) · **Q1.4** value cleaning: HTML, translations,
  the 500-character cap, invisible characters (A5, SK3) · **Q1.5** stemming/tokenisation, accents
  (A3, SA12) · **Q1.7** type names and usernames as words (SA9) · **Q1.9** Typesense version
  (A2) · **Q1.12** complete-indexed fields (SK3) · **Q1.13** the 500-character cap (SK3) · **Q1.14**
  the registry as the extension contract (B2) · **Q1.15** the `0` sentinel in the node arrays (B3)
- **Q2.1** which keyword quirks to copy, decline, improve (SA*) · **Q2.4** stop words (SA2, SA3,
  SB5) · **Q2.5** phrases under stemming (SA3; moot if Q2.14 pre-stems) · **Q2.6** negated phrase
  (SA4) · **Q2.7** OR groups (SA7) · **Q2.8** punctuation and commas (SA6, SB4) · **Q2.9** type-name
  words (SA9) · **Q2.10** related keywords (SA10) · **Q2.11** bare years and numbers (SA9) ·
  **Q2.12** unknown words (SA11) · **Q2.13** quirks to raise upstream · **Q2.15** full-text box veto
  (SA8) · **Q2.16** short wildcards (SA5) · **Q2.17** wildcard inside a word (SA5) · **Q2.18**
  accent folding (SA1)
- **Q3.2** quoted field terms (SB2) · **Q3.3** `basic*` (SD2) · **Q3.5** date-range fields (SD4) ·
  **Q3.6** non-indexed fields (SK1, SD5) · **Q3.7** numeric edge cases (SE1) · **Q3.8** field
  negation (SB5) · **Q3.9** node limits (SC1) · **Q3.10** field wildcard caps (SB3) · **Q3.11** /
  **Q3.17** field-name rules (SB1) · **Q3.13** date components (SD2) · **Q3.14** two values and `;`
  on a date (SD1) · **Q3.15** `$ignore_filters` (D2) · **Q3.16** dates/numbers regardless of flag
  (SD5) · **Q3.18** phrase or AND in a quoted field value (SB2, SB4)
- **Q4.1** which specials, in what order (SF9–SF16) · **Q4.2** fields in one batch (SF9–SF16) ·
  **Q4.4** `restypes` on specials (SF5, SF6, D5) · **Q4.5** workflow states per special (SF7) ·
  **Q4.6** smart collections (SF2) · **Q4.7** `!unused` (SF16) · **Q4.8** `!empty` (SF10) · **Q4.9**
  unknown specials (SF8) · **Q4.10** `!properties` + `!list` (SF9, SJ1) · **Q4.11** property keys
  (SF9)
- **Q5.1** collection gate helper or copy (SF2, I2) · **Q5.2** collection columns at hydrate (D11) ·
  **Q5.3** relevance inside a collection (SF2) · **Q5.4** upload-share sessions (SF2) · **Q5.5**
  searching *for* collections in scope? · **Q5.6** collection freshness mechanism (C6, B5) · **Q5.7**
  which collections are indexed (B5) · **Q5.8** `J` with `j*` (D8) · **Q5.9** reordering from the
  search page (SF2) · **Q5.10** large collections and `fetchrows = -1` (D10) · **Q5.11**
  `$smartsearch` (D5)
- **Q6.4** editable-only (D1) · **Q6.5** Typesense-only and access (D1) · **Q6.6** `$heightmin`
  (D5) · **Q6.7** NOT over hidden/inactive fields (SA4, SK2) · **Q6.8** share keys and external
  access (SF2, D6) · **Q6.9** access config matrix (D5) · **Q6.10** validation in Phase 0 (C8)
- **Q7.2** further sorts (SG1–SG4) · **Q7.4** exact ref first (SA9, SF4) · **Q7.5** page offset with
  collections (D10) · **Q7.6** row shape (D11) · **Q7.7** deep paging (D10) · **Q7.8** sort
  direction quirks (D9) · **Q7.9** `field<n>` on non-join columns (SG4)
- **Q8.8** grants and memberships interim (C6) · **Q8.9** hit-count freshness (SG2) · **Q8.10**
  rebuild window and capacity (B8) · **Q8.11** topology (A2) · **Q8.12** observability (G1) ·
  **Q8.13** which register rows must be immediate in M3 (C5, C6, C9)
- **Q9.3** rebuild and resync from the UI (G1) · **Q9.4** performance work now or after sync (B8) ·
  **Q9.5** logging (G2) · **Q9.6** health, warnings and the badge (G1, G3) · **Q9.7** deployment and
  versioning (G5) · **Q9.8** several installations on one server — *decided 2026-10-14: required* ·
  **Q9.9** test hosts (H4) · **Q9.13** shared-server limits and schedules (G6, B8)
- **Q10.2** CI (H1) · **Q10.5** credentials (H4) · **Q10.6** freshness tests local (H5) · **Q10.7**
  performance tests (H6) · **Q10.8** live user matrix (H4) · **Q10.9** cadence (H) · **Q10.10**
  core-bug batteries (H7) · **Q10.14** API-unreachable rows (D1, D11, SK2, SK3)
- **Q11.3** each core bug (I4) · **Q11.4** tracking upstream (I5) · **Q11.5** a cleaner provider
  contract (I1) · **Q11.6** upstream intent (I*)
- **Q12.5** erasure and retention (X5) · **Q12.7** logging and redaction (X1, G2, G7) · **Q12.8**
  keys per instance (X1, G6) · **Q12.9** supply chain (A7, X6) · **Q12.10** the standing security
  cases (X4)

## Standing after the marks (2026-10-20)

Every row of the catalogue is marked, so each question above carries a *Standing* line: **required** when a
*must* or *bug* row cites it or it blocks a task that is not parked; **check-only** when only *check* rows
cite it and it blocks nothing else — it can wait for the check; **pruned** when only *no* rows cite it and
it blocks only parked packages; **uncited** when no row and no blocking line names it (it acts through its *Shapes* line). For Part 3:

### Check-only

- **Q1.12** — check rows K9
- **Q2.17** — check rows A13
- **Q6.4** — check rows G38

### Pruned

- none: every Part 3 question is cited by a required row or blocks a live task

### Cited by no row and blocking no task

Q0.10 · Q0.11 · Q0.12 · Q1.14 · Q2.18 · Q5.5 · Q8.11 · Q9.8 · Q10.14 — answered when their task starts; nothing in the catalogue turns on them.

Everything else in Part 3 is required.

### Part 3 questions decided

- **Q1.9** (2026-10-20): Typesense 30.0 or later, tested on 30.x; refuse below, warn above the tested range. decisions.md.
- **Q9.13** (2026-10-21): the operator manages the shared server; the plugin is tenant-local — staggered windows by cron, two import streams by default, limits mirrored from the checklist, own figures only on the page; tenant keys only (Q12.11 direction). decisions.md.
- **Q1.3** (2026-10-21): fragments stay and are free-text searchable (`query_by`), as in core; K5 served by SB6. decisions.md.
- **Q1.4** (2026-10-21): value cleaning does what core does at each site — tags stripped as core strips them, every translation indexed, the 500-character cap, invisible characters stripped. decisions.md.
- **Q10.2** (2026-10-21): no CI job; both runners kept; a one-file `tests/` entry runs the smoke battery under `composer dev:beforeCommit` when a local Typesense is reachable. decisions.md.
- **Q12.5** (2026-10-22): the index is derived — no backups or snapshots, rebuilt from the database; delete paths as designed; Typesense logs off or short-rotated; plugin debug via core's log; sync log thirty days; harness never on real data. decisions.md.
- **Q12.7** (2026-10-22): logging as designed — keys and headers redacted, sync log refs and outcomes only, the per-search debug line behind `$debug_log` under core's policy, the explain tool admin-only and never a key. decisions.md.
- **Q12.8** (2026-10-22): one scoped pair per tenant; aliases kept with a narrow grant (create / get / delete, no list) and prefixed, unguessable names; the same pattern allowed for synonym sets (Q2.10 chooses); a serve-time identity guard; X1's alias cases decide between this and one Typesense process per tenant. decisions.md.
- **Q12.9** (2026-10-22): settled by Q9.10 — no dependency, no outbound call except Typesense, both confirmed in X6; who tracks Typesense advisories is for the meeting. decisions.md.
- **Q12.10** (2026-10-22): X4 as written plus the cross-tenant join read, the identity guard, provisioning, the gate uses after I1, the three share paths and an excluded field. decisions.md.
