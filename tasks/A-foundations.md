# A — Foundations

Milestone M0. Everything else builds on these five modules.

### A1 — Plugin skeleton
- **Builds:** `typesense_search.yaml`, `config/config.php` (defaults only), `languages/en.php`,
  `hooks/all.php` (registrations, no logic), `pages/setup.php` (shell).
- **Size:** S · **Depends on:** —
- **Must honour:**
  - A group-restricted plugin's hooks run only for those groups and never in cron; the sync worker
    and the rebuild must see the plugin → `disable_group_select: 1` (global) with a per-group
    `$typesense_search_enabled` override for A/B groups (P13).
  - Whether a per-group config override is applied before the search hook runs is untested (Q8.3):
    this task includes that test.
  - The collection prefix is generated at first setup and stored (Q9.12 decided (b), 2026-10-21),
    never entered; the old default was `test3_`.
  - **Options are read through one accessor** (Q9.1, 2026-10-21): `$typesense_search_<option>_override`
    in `config.php` wins over the stored value, which wins over the default; the setup page draws an
    overridden option locked ("locked in global configuration to: [value]"), keys as "(hidden)". The
    editable set, the read-only health block, the lockable actions and the `config.php`-only knobs
    are listed in decisions.md (Q9.1).
  - Comments terse, in the plugin's existing style (P36).
- **Blocking questions:** Q0.13 (decided), Q0.14 (decided), Q9.1 (decided), Q9.2 (superseded by Q12.1, decided)
- **Proves it:** plugin activates on a clean install; setup page renders; a per-group override
  changes behaviour in a request made by a user of that group (the Q8.3 test).

### A2 — Typesense client
- **Builds:** a curl client (Q9.10 decided (b), 2026-10-20) behind one transport seam. `include/client.php`
  holds the contract every caller uses — `search()` (GET or POST `multi_search` by size),
  `multi_search()`, `import()` with per-line results, `delete()`, `collection_exists()`, collection,
  alias and health calls — and everything transport-independent: the size routing, the chunking, the
  ready flag and circuit breaker, the two timeouts, exceptions mapped to the breaker so nothing leaks
  into a search. `include/client_curl.php` is the transport (keep-alive handle, `curl_multi` for
  parallel imports). A later `client_library.php` implements the same contract if the official
  library is adopted, with no change to callers; this battery is the swap test.
- **Size:** M · **Depends on:** A1
- **Must honour:**
  - A GET query string is capped at 4,000 bytes; `per_page` above 250 returns 422; `multi_search`
    takes at most 50 searches; `page` can go deep but cost grows faster than linear.
  - `update` on a missing id fails per line inside an HTTP 200: check every JSONL line and report
    each failure (resource failures cascade into membership and grant reference failures).
  - Typesense rejects a numeric string (`"5"`) for an int field.
  - Separate timeouts for search and import (the schema upgrade needed 600 s); one keep-alive handle.
  - On 503 back off 10–60 s with jitter and halve concurrency (P16); concurrent imports ≤ cores − 2;
    leave the server's own batch size alone.
  - Never create collections outside the rebuild path.
  - **Server version** (Q1.9 decided (a), 2026-10-20): minimum 30.0, tested on 30.x — read once at setup
    from `/debug`; below the minimum the plugin refuses to activate, above the tested range it warns;
    the battery records the tested range.
  - An outage must end in a fallback to MySQL, never an empty result (P7 rollback = fall back).
  - **Breaker** (Q8.6 decided, 2026-10-20): trips on transport failures and 5xx only, never on a 4xx;
    open = immediate fallback with no network call; half-open probe against `/health` after a
    cooldown; state readable by the setup page (G1) and by the flush (C3), which skips while open.
- **Blocking questions:** Q8.6 (decided), Q9.1 (decided), Q9.10 (decided), Q1.9 (decided), Q12.1 (decided), Q12.2 (decided)
- **Proves it:** a client battery on the local Typesense: window shapes, 422 and 400 handling, import
  line errors surfaced, breaker trips on repeated failures and resets, GET/POST choice by size.

### A3 — Schema module
- **Builds:** `include/schema.php` — `schema_version()`, `resources_schema()`, `memberships_schema()`,
  `grants_schema()`, alias and generation naming, `token_separators()`.
- **Size:** M · **Depends on:** A1
- **Must honour:**
  - Text fields stem iff `$stemming`; `_q` representation arrays never stem (P31).
  - `token_separators` = `$config_separators` plus `,` and `:`, single-byte only (Typesense rejects
    curly quotes); space excluded.
  - A reference may name an alias but binds to the real collection at creation; memberships and
    grants must be created referencing the generation's resources collection, and all three aliases
    swapped together (moving only the resources alias breaks joins).
  - Three sort fields maximum (docs 30.0: "Up to 3 sort fields can be specified … If all 3 fields tie,
    the document insertion order is used"); a string field sorts only with `sort: true` and "requires
    the construction of a separate index that can consume a lot of memory for long string fields";
    missing values: numeric "always sorted to the end regardless of the sort order", string empty or
    missing "the highest value" (end ascending, top descending), and `field(missing_values: first|last)`
    overrides both — an earlier live note said the option did not exist: verify on 30.2 in A3's
    battery. `_eval(<filter expression>)` and `_rand([seed])` are sort expressions in 30.0.
  - **Composite sort keys** (Q7.2): a core sort with more than three keys is one fixed-width key built
    by B2 at index time, valid because core applies one direction to every key of a sort; each part
    encodes "no value" explicitly (`0:`/`1:`/`2:` as the date key does) because missing-value
    placement differs from MySQL's; the last part is always `ref` so a tie never reaches insertion
    order.
  - `created_by` optional (NULL for CLI-created resources); `created_date` was replaced by
    `creation_date` (day limit) and `date_field_sort` (text sort key) (P19, P20).
  - Dynamic field patterns `field_.*_<suffix>`; explicit fields where a per-field option is needed.
  - Per-collection synonyms are gone in 30.x; synonym sets instead.
  - **Every active field's values are indexed** (Q1.1 decided (b), 2026-10-20): the index flags decide
    `query_by` membership and the word filters at query time, not presence in the document; the
    condition is a rebuild on the test system with field 86 (`accessibilityalttext`, 25.7 MB of text)
    included, measured against the 800 MB baseline; a field on the exclusion list (G9) gets nothing.
  - **Version 1 of the schema** (Q1.2 decided (a), 2026-10-20; amended by Q3.4, 2026-10-21): `_s`,
    `_text`, `_ss`, `_q` for dates and numbers only, `_f`, `_k` (the fourteen-digit date key), `_p`; no
    `_q` on text fields, no `_ts`, no facet on `_s`, no range pair. A representation enters the schema only with the package that proves it, and additions
    are batched into one rebuild per release (hours on a 1.4 M-resource system).
  - **Version 1 top-level fields** (Q1.6 decided (b), 2026-10-20): the base rows of the register plus
    `hit_count`, `user_rating`, `rating` and the four packed sort keys — confirmed by Q7.1 (B); every other
    candidate arrives with its package.
- **Blocking questions:** Q1.1 (decided), Q1.2 (decided), Q1.6 (decided), Q1.8 (decided), Q1.10 (decided), Q1.11 (decided), Q7.1 (decided)
- **Proves it:** collections created from scratch on the local Typesense; a schema snapshot test
  against the schema document; sample documents for every field pattern accepted.

### A4 — Date parsing
- **Builds:** `include/dates.php` — `parse_date()` returning the representations, the fourteen-digit
  comparison key and the precision (Q3.4 decided (a), 2026-10-21: no range start and end in version
  1); `range_bounds()` building a query's start and end keys the way core builds its strings (the end
  padded to 235959); `date_sort_key()`; `timestamp()` for the resource-level epochs.
- **Size:** S · **Depends on:** —
- **Must honour:**
  - Forms: `YYYY`, `YYYY-MM`, `YYYY-MM-DD`, with optional time after a full date; RS uses `00` for an
    unknown month or day (`2024-00-00`, `2024-08-00`, `0000-08-00`, `0000-00-11`).
  - Year and month precision give an interval but no timestamp; month-only and day-only give
    representations only (no position on the timeline).
  - The advanced form pads a range end with `-31` and EDTF uses `-99`: `2024-02-31` is not a real
    date but core compares it as a string, so it must still bound a range — the fourteen-digit key
    does (Q3.4 decided (a)): digits padded to `YYYYMMDDHHMMSS`, non-real dates kept, a non-digit
    value gets no key and never matches.
  - Date sort key: `0:` NULL, `1:` empty, `2:<value>` reproduces core's string sort of the cache
    column; matching the date field needs a word boundary (field 120 is not field 12).
  - Epochs (`_ts`, `creation_date`, `modified_date`) and the day-limit cut-off use the time zone of
    the MySQL values: RS stores server-local time, so PHP's default zone and MySQL's session zone
    must agree — a battery case at a DST change and at midnight.
- **Blocking questions:** Q3.4 (decided)
- **Proves it:** a unit table of every form above, the padded and EDTF ends, invalid dates, and the
  sort key.

### A5 — Text normalisation
- **Builds:** `include/text.php` — `clean()`, `strip_markup()`, `translations()`, `cap()`,
  `fragments()`.
- **Size:** S · **Depends on:** —
- **Must honour:**
  - Strip invisible characters: `\p{Cf}` removed, `\p{Zs}` → space (6,548 documents on the test
    system held a BOM-prefixed option; RS's `cleanse_string` strips it, Typesense kept it) (P33).
  - Raw HTML makes tag names searchable and glues the word after a tag; translated values
    `~en:A~fr:B` glue to `afr`; core indexes the first `$node_keyword_index_chars` (500) characters.
  - **Do what core does at each site** (Q1.4 decided (a), 2026-10-21), calling core's functions where
    they can be called: tags stripped as `node_functions.php` strips them before keyword mapping
    (`<a>` kept, so link text and URL words are searchable, as in core); every translation indexed;
    the 500-character cap on the keyword representations, the uncapped `_w` only if K10 requires it;
    invisible characters stripped as `cleanse_string` does (P33).
  - Fragments: every substring ≥ `$partial_index_min_word_length` of each word, lower-cased, as
    `add_partial_index()` does; mean 49 per filename, ~108 MB for 114,596 filenames.
- **Blocking questions:** Q1.4 (decided), Q1.3 (decided)
- **Proves it:** unit cases for BOM/NBSP/word-joiner, `<p>hello`, a link whose URL words core indexes,
  `~en:A~fr:B` with a phrase spanning the two translations, the 500-character cap, and fragment sets
  — each compared with core's own function on the same sample values.

### A6 — Plugin state store
- **Builds:** `include/state.php` over a plugin table `typesense_search_state` (prefix, name, value,
  updated; `dbstruct`, created on activation like the queue table — Q9.11 decided (b), 2026-10-21):
  `state_get(name, default)`, `state_set(name, value)` as one insert-or-update,
  `state_increment(name)` in place for the counters; the named entries every other
  task reads: `schema_version` of the live generation, `live_generation`, `building_generation`,
  `ready`, `bulk_pending`, `last_rebuild` (time, result, duration, counts), `last_worker_run`,
  `breaker_open_until`, `field_snapshot`, `config_snapshot`, and the counters (validation drops,
  parked rows, breaker trips); keyed by collection prefix so several instances on one server stay
  apart (G6).
- **Size:** S · **Depends on:** A1
- **Must honour:**
  - Written by CLI processes (the rebuild, the worker) and by web requests alike, so it lives in
    MySQL, never in a file or APCu, and never in `sysvars`: its setter is a delete-then-insert, a
    null-equal value deletes the row, and its per-process array and query cache serve stale reads
    to a long-running process (Q9.11); structured values as JSON; every read goes to the table
    (a per-request cache with a short TTL at most).
  - `ready` and the breaker are read on every search: a per-request cache with a short TTL is
    fine, but the flag must survive a PHP-FPM restart and must default to *not ready* on a fresh
    install.
  - Plugin configuration (`set_plugin_config`) is for options, not state; the two never mix.
- **Blocking questions:** Q9.11 (decided)
- **Proves it:** a unit battery (set, get, JSON round trip, a counter incremented from two processes,
  prefix isolation); B8, B9, C4, C7 and G1 read and write state only through it.

### A7 — Vendored dependencies
- **Dropped from M0 (Q9.10 decided (b), 2026-10-20):** the client is custom and the plugin has no
  dependency to vendor. Kept as the procedure for adopting the official library later behind A2's
  transport seam; not in any milestone until then.
- **Builds:** `plugins/typesense_search/lib/` holding the official `typesense-php` client and its
  HTTPlug stack, installed by Composer from a committed `lib/composer.json` and `composer.lock`,
  its `vendor/autoload.php` required once from `hooks/all.php`; `lib/README.md` with the pinned
  versions, the licence notices, the server versions verified against, and the upgrade procedure
  (`composer update` in `lib/`, the A2 battery, a plugin version bump).
- **Size:** S · **Depends on:** A1 · only when Q9.10 chooses the library
- **Must honour:**
  - The plugin ships inside ResourceSpace 11.1 (2026-10-20), so `lib/` is committed into the
    ResourceSpace repository itself: its licences and supply chain are reviewed under Montala's own
    policy (Q12.9); `phpcs.xml.dist` already excludes `plugins/*/lib`.
  - Core's own `vendor/` is not tracked and is installed at deploy time; the plugin's `lib/` is
    committed complete so a deployment needs no Composer step on the host (as `simplesaml`, `totp`
    and `adobe_link` do); `phpcs.xml.dist` already excludes `plugins/*/lib`.
  - Pass the PSR-18 client to the Typesense client explicitly so HTTPlug discovery does not run on
    every web request.
  - Pin a stable release (6.1.0-RC2 is a release candidate); PHP 8.2 is core's floor, the
    library's is 7.4.
  - A package core already ships (`psr/http-message`, Guzzle through `google/apiclient`) must not
    be loaded twice in different versions — check the deployed core's `vendor/` and prefer the
    `php-http/curl-client` adapter, which pulls nothing core has.
- **Blocking questions:** Q9.10, Q12.9
- **Proves it:** a clean checkout with the plugin enabled loads the client with no Composer step on
  the host; the A2 battery passes through it; `composer validate` and `composer audit` are clean in
  `lib/`.

### A8 — Tenancy
- **Builds:** `include/tenancy.php` — the prefix as the single tenant boundary: generated at first
  setup as a random token (Q9.12 decided (b), 2026-10-21), stored, read-only on the page, overridable
  from `config.php`; validation of the override (letters, digits, underscore, a length cap); a lockable
  "regenerate prefix" action for the clone case; `name(kind, generation)` producing every
  server-side name (`<prefix>_resources_g<N>`, the aliases, the synonym set), the installation
  identity written into each generation's metadata (`$baseurl`, database name, plugin version)
  and the clash check that refuses to build over another installation's collections; the state (A6)
  and the queue (C1) already partition by prefix and database.
- **Size:** S · **Depends on:** A1, A3
- **Must honour:**
  - Nothing may address a collection, alias or synonym set except through `name()`; the harness
    uses its own prefix on the shared local server for the same reason.
  - The old default prefix (`test3_`) would have made two installations share an index; there is
    no default now.
  - Server-side limits are shared: the filter-operations guard, memory, CPU, import concurrency —
    Q9.13 decided (a), 2026-10-21: the operator staggers the rebuild windows by cron, import
    concurrency defaults to two streams and is raised per tenant from `config.php`, the
    filter-operations limit is mirrored from G8's checklist, and the page shows the tenant's own
    figures only; a tenant never assumes it is alone and never holds a server-wide key.
  - Keys are per tenant and scoped to `<prefix>_*` (X1, Q12.8) — and the pattern confines
    collection endpoints only, so aliases and synonym sets rely on narrow grants and unguessable
    names (Q12.8 decided 2026-10-22): a `config.php` prefix override is validated as not being a
    prefix of, or prefixed by, any other tenant's on the server at provisioning; **a serve-time
    guard** — before serving, and after every rebuild — checks that the alias resolves to a
    generation whose metadata carries this installation's identity, and otherwise falls back and
    alerts; X1's alias cases decide whether this is enough or whether a shared host runs one
    Typesense process per tenant.
- **Blocking questions:** Q9.12 (decided), Q9.13 (decided), Q12.8 (decided)
- **Proves it:** `harness/ab/A8_tenancy.php` — two prefixes on the local server: each rebuilt,
  searched, resynced and deleted without touching the other; a prefix clash refused; an invalid
  prefix refused; every generated name matches the pattern.
