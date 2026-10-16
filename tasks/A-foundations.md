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
  - The collection prefix is mandatory and validated — no default at all (A8); the old default
    was `test3_`.
  - Comments terse, in the plugin's existing style (P36).
- **Blocking questions:** Q0.13, Q0.14, Q9.1, Q9.2
- **Proves it:** plugin activates on a clean install; setup page renders; a per-group override
  changes behaviour in a request made by a user of that group (the Q8.3 test).

### A2 — Typesense client
- **Builds:** `include/client.php` — either a thin adapter over the official `typesense-php` client
  vendored in `lib/`, or a curl client (Q9.10): `search()` (GET or POST `multi_search` by size),
  `multi_search()`, `import()` with per-line results, `collection_exists()`, alias calls; a ready
  flag and a circuit breaker; exceptions mapped to the breaker so nothing leaks into a search.
- **Size:** M · **Depends on:** A1, A7 (when Q9.10 chooses the library)
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
  - An outage must end in a fallback to MySQL, never an empty result (P7 rollback = fall back).
- **Blocking questions:** Q8.6, Q9.1, Q9.10, Q1.9, Q12.1, Q12.2
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
- **Blocking questions:** Q1.1, Q1.2, Q1.6, Q1.8, Q1.10, Q1.11, Q7.1
- **Proves it:** collections created from scratch on the local Typesense; a schema snapshot test
  against the schema document; sample documents for every field pattern accepted.

### A4 — Date parsing
- **Builds:** `include/dates.php` — `parse_date()` returning representations, timestamp, range start
  and end, precision; `date_sort_key()`; `timestamp()`.
- **Size:** S · **Depends on:** —
- **Must honour:**
  - Forms: `YYYY`, `YYYY-MM`, `YYYY-MM-DD`, with optional time after a full date; RS uses `00` for an
    unknown month or day (`2024-00-00`, `2024-08-00`, `0000-08-00`, `0000-00-11`).
  - Year and month precision give an interval but no timestamp; month-only and day-only give
    representations only (no position on the timeline).
  - The advanced form pads a range end with `-31` and EDTF uses `-99`: `2024-02-31` is not a real
    date but core compares it as a string, so it must still bound a range (Q3.4 decides how).
  - Date sort key: `0:` NULL, `1:` empty, `2:<value>` reproduces core's string sort of the cache
    column; matching the date field needs a word boundary (field 120 is not field 12).
  - Epochs (`_ts`, `creation_date`, `modified_date`) and the day-limit cut-off use the time zone of
    the MySQL values: RS stores server-local time, so PHP's default zone and MySQL's session zone
    must agree — a battery case at a DST change and at midnight.
- **Blocking questions:** Q3.4 (whether a fixed-width comparison key is also needed)
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
  - Fragments: every substring ≥ `$partial_index_min_word_length` of each word, lower-cased, as
    `add_partial_index()` does; mean 49 per filename, ~108 MB for 114,596 filenames.
- **Blocking questions:** Q1.4, Q1.3
- **Proves it:** unit cases for BOM/NBSP/word-joiner, `<p>hello`, `~en:A~fr:B`, the 500-character
  cap, and fragment sets against RS's own function on sample words.

### A6 — Plugin state store
- **Builds:** `include/state.php` — `state_get(name, default)`, `state_set(name, value)`,
  `state_update(name, fn)` (a locked read-modify-write for counters); the named entries every other
  task reads: `schema_version` of the live generation, `live_generation`, `building_generation`,
  `ready`, `bulk_pending`, `last_rebuild` (time, result, duration, counts), `last_worker_run`,
  `breaker_open_until`, `field_snapshot`, `config_snapshot`, and the counters (validation drops,
  parked rows, breaker trips); keyed by collection prefix so several instances on one server stay
  apart (G6).
- **Size:** S · **Depends on:** A1
- **Must honour:**
  - Written by CLI processes (the rebuild, the worker) and by web requests alike, so it lives in
    MySQL, never in a file or APCu; core's `sysvars` table with `get_sysvar()` / `set_sysvar()` is
    the obvious home under a `typesense_search_<prefix>_` key, a plugin table the alternative
    (Q9.11); structured values as JSON.
  - `ready` and the breaker are read on every search: a per-request cache with a short TTL is
    fine, but the flag must survive a PHP-FPM restart and must default to *not ready* on a fresh
    install.
  - Plugin configuration (`set_plugin_config`) is for options, not state; the two never mix.
- **Blocking questions:** Q9.11
- **Proves it:** a unit battery (set, get, JSON round trip, a counter incremented from two processes,
  prefix isolation); B8, B9, C4, C7 and G1 read and write state only through it.

### A7 — Vendored dependencies
- **Builds:** `plugins/typesense_search/lib/` holding the official `typesense-php` client and its
  HTTPlug stack, installed by Composer from a committed `lib/composer.json` and `composer.lock`,
  its `vendor/autoload.php` required once from `hooks/all.php`; `lib/README.md` with the pinned
  versions, the licence notices, the server versions verified against, and the upgrade procedure
  (`composer update` in `lib/`, the A2 battery, a plugin version bump).
- **Size:** S · **Depends on:** A1 · only when Q9.10 chooses the library
- **Must honour:**
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
- **Builds:** `include/tenancy.php` — the prefix as the single tenant boundary: validation
  (mandatory, letters, digits, underscore, a length cap), `name(kind, generation)` producing every
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
  - Server-side limits are shared: the filter-operations guard, memory, CPU, import concurrency
    (Q9.13); a tenant never assumes it is alone.
  - Keys are per tenant and scoped to `<prefix>_*` (X1, Q12.8); whether alias and synonym-set
    operations are also scoped is verified in X1's battery (Q12.11).
- **Blocking questions:** Q9.12, Q9.13, Q12.8
- **Proves it:** `harness/ab/A8_tenancy.php` — two prefixes on the local server: each rebuilt,
  searched, resynced and deleted without touching the other; a prefix clash refused; an invalid
  prefix refused; every generated name matches the pattern.
