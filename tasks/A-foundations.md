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
  - Default collection prefix must not be a test value (the old default was `test3_`).
  - Comments terse, in the plugin's existing style (P36).
- **Blocking questions:** Q0.13, Q0.14, Q9.1, Q9.2
- **Proves it:** plugin activates on a clean install; setup page renders; a per-group override
  changes behaviour in a request made by a user of that group (the Q8.3 test).

### A2 — Typesense client
- **Builds:** `include/client.php` — either a thin adapter over the official `typesense-php` client
  vendored in `lib/`, or a curl client (Q9.10): `search()` (GET or POST `multi_search` by size),
  `multi_search()`, `import()` with per-line results, `collection_exists()`, alias calls; a ready
  flag and a circuit breaker; exceptions mapped to the breaker so nothing leaks into a search.
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
  - An outage must end in a fallback to MySQL, never an empty result (P7 rollback = fall back).
- **Blocking questions:** Q8.6, Q9.1, Q9.10
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
  - Three sort fields maximum; string sort is case- and accent-insensitive; an empty string is
    "missing"; missing sorts last ascending and first descending; no `missing_values` option.
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
