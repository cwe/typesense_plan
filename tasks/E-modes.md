# E — Modes (special searches)

Milestone M5 for E1–E9; E10 onwards in M7 as their index fields arrive. One mode per task. Every mode
gets keyword and node matching from D4 automatically.

### E1 — Standard mode
- **Builds:** `modes/standard.php` (a marker: no scope of its own).
- **Size:** S · **Depends on:** D4
- **Proves it:** the plain keyword battery (D4) passes through it.

### E2 — Collection mode
- **Builds:** `modes/collection.php`, `include/collections.php` (the gate).
- **Size:** M · **Depends on:** D4, D8, B3, I2 (if G1)
- **Must honour:**
  - Gate: core's valid-collection list (own, public, featured through the access control, request,
    research, upload-share session collections, the share key, `$ignore_collection_access`), then
    `collection_readable()`; never skipped under `access_override`; negative refs fall back.
  - Collection id parsed as core does: split on space, then comma, then `(int)`.
  - Join filter `$memberships(collection_ref:=C)`; default-archive restriction suppressed;
    `archive:!=2` when `$collections_omit_archived` and no `e2`, only without `access_override`;
    the `$k` pending exemption (D5).
  - Default order `$memberships(sortorder:d, date_added:opposite),ref:d` (verified on three
    collections both ways); an explicit sort goes through D9; relevance inside a collection per
    Q5.3; `restypes` are ignored for collections (core).
  - Volatile collections (selection, upload, share-upload) veto; the selection bar's search is one
    of them.
  - Smart collections: refresh only after the plugin has committed to serving, with core's 2-hour
    limit for featured smart collections, or veto them (Q4.6).
  - `!collection158` errors in core on the test system — unexplained; note it.
- **Blocking questions:** Q5.1, Q5.2, Q5.3, Q5.4, Q5.8, Q4.6
- **Proves it:** the 13-case collection battery and the parity plan's Phase 2 matrix (users ×
  collections × queries × `fetchrows`).

### E3 — Last mode
- **Builds:** `modes/last.php`.
- **Size:** S · **Depends on:** D10
- **Must honour:**
  - Core: inner `ORDER BY ref DESC LIMIT n`, outer `ORDER BY` the user's sort → recent selection
    (D10) plus the user's sort; a non-numeric argument (`!last`, `!lastabc`, `!last100 word` with a
    space) means 1000, the comma form keeps N (P28); keyword and node criteria still apply.
  - Relevance → `ref` with the upper-case-DESC test (P29).
  - Core bug: `!last` + a node bucket returns one row (Core 1) — serve correctly, replicate or veto
    per Q11.3.
- **Blocking questions:** Q4.3, Q11.3
- **Proves it:** the 11-case last battery.

### E4 — List mode
- **Builds:** `modes/list.php` (`!list`, `!listall`).
- **Size:** S · **Depends on:** D4
- **Must honour:** colon-separated refs from the first token, non-integers dropped, none valid →
  nothing (`ref:<0`); any archive state; `!listall` is identical to `!list` in core.
- **Blocking questions:** Q4.3
- **Proves it:** list battery.

### E5 — Resource mode
- **Builds:** `modes/resource.php` (`!resource<ref>`, bare numbers with `$config_search_for_number`).
- **Size:** S · **Depends on:** D4, D9
- **Must honour:** core takes every digit of the string (`!resource416, date:2024` → 4162024, Core 3);
  with the option off, a bare number is a keyword search with the exact ref first (D9, Q7.4);
  `get_resource_access()` uses this search as a gate (Q6.2).
- **Blocking questions:** Q4.3, Q2.11, Q6.2
- **Proves it:** resource battery.

### E6 — Contributions mode
- **Builds:** `modes/contributions.php`.
- **Size:** S · **Depends on:** D4, D6
- **Must honour:** `created_by:=U`; for the user's own with `$open_access_for_contributor`, core drops
  every filter and join (keyword criteria included) and zeroes the grant columns → suppress all
  restrictions and keyword matching, `ref:>0`, `archive:=[raw states cast to int, "" → 0]`, veto on
  an empty list, zeroed select at hydrate (P26). The user-actions page uses it for submit-pending
  (state −2).
- **Blocking questions:** — (confirm P26)
- **Proves it:** the 17-case contributions battery.

### E7 — Hasdata mode
- **Builds:** `modes/hasdata.php`.
- **Size:** S · **Depends on:** D4
- **Must honour:** `populated_field_ids:=[ref]`; every workflow state; works for non-indexed,
  inactive and hidden fields because `populated_field_ids` comes from all nodes.
- **Proves it:** hasdata battery (indexed, non-indexed, inactive, hidden field).

### E8 — Pending modes
- **Builds:** `modes/pending.php` (`!archivepending` = 1, `!userpending` = −1).
- **Size:** S · **Depends on:** D4
- **Must honour:** the state parameter is ignored; `!archivepending` returns nothing for a user with
  `z1` in core (Q4.5); `!userpending` with rating sort orders by `request_count` in core (veto).
- **Blocking questions:** Q4.5
- **Proves it:** pending battery for a `z1` user and a `v` user.

### E9 — Unsupported mode
- **Builds:** `modes/unsupported.php` — vetoes every other `!command`, including unknown and
  misspelt ones.
- **Size:** S · **Depends on:** D2
- **Must honour:** core returns **everything** for an unknown special; the plugin's veto hands the
  search to core, so the user sees that either way (Q4.9).
- **Blocking questions:** Q4.9
- **Proves it:** every catalogue group F row not served falls back.

### E10+ — New modes, one task each, in the order Q4.1 decides
Each needs its index fields (Q1.6, B1/B2) and a rebuild. Candidates with what they need:
- `!related` / `!relatedpushed` — related refs; both directions; `$related_search_show_self`; core's
  push-metadata type join. (`!related<ref> word` exceeded 120 s in core.)
- `!properties` — width, height, file size, extension, `has_image`, `created_by`, orientation;
  `;`-separated AND; combines with keywords (space-separated); the form's `!list` collision (Q4.10).
- `!images`, `!nopreview` — `has_image`.
- `!locked`, `!integrityfail`, `!noningested` — `lock_user`, `integrity_fail` + `no_file`,
  `file_path`; `!noningested` is admin-only and unfiltered in core.
- `!geo` — a geopoint; core's encoded bounding box.
- `!colour`, `!colourkey` — `colour_key` prefix and 4-character key with `has_image`.
- `!unused` — absence of any membership (verify Typesense can express it, Q4.7).
- `!empty<field>` — `populated_field_ids:!=[ref]` (verified 112,596); core's own `!empty` is buggy
  (Core 5, 6) — Q4.8.
- Permanently on core: `!rgb`, `!duplicates`, `!nodownloads`, `!report`.
