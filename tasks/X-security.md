# X — Security and data protection

Seven tasks from [chunk 12](../chunks/12-security-and-data-protection.md). X1–X3 are milestone M0
(nothing talks to a server before keys, transport and web exposure are right), X4 is M2 (it runs
with the first served search and on every change after), X5 is M3, X6 is the review before the
pilot (M5), X7 is M7 with the documentation. The letter is X because J and K are catalogue groups.

A failing case in X4 blocks a merge whatever the accepted-divergence list says: a security
difference is never an accepted difference.

### X1 — Key handling
- **Builds:** (Q12.1 decided 2026-10-22) two keys per installation — `typesense_search_search_key`
  (search on `<prefix>_*`, the only key the request path loads) and `typesense_search_admin_key`
  (documents, collections and aliases on the prefix, loaded only by the rebuild and the worker) —
  entered on the setup page as password fields and stored as plugin options, with
  `$typesense_search_search_key_override` / `_admin_key_override` in `config.php` winning and hiding
  them (Q9.1's lock rule); the client (A2) picking the key by operation; the page showing "set" /
  "not set" and never the value, not even in the HTML source, and "locked in global configuration
  (hidden)" when locked; a provisioning action that takes the server's bootstrap key once, creates
  both scoped keys with the right actions and pattern, stores them and never stores or logs the
  bootstrap key; rotation = the same action (create, switch, revoke the old); redaction of keys and
  request headers in every log line.
- **Size:** S · **Depends on:** A1, A2
- **Must honour:** a Typesense key is scoped by actions and by collection pattern — the search key
  gets `documents:search` on `<prefix>_*` and nothing else, and both keys are per installation
  (Q12.8, A8). **The pattern confines collection endpoints only** (docs 30.0: it "does not restrict
  global endpoints like synonym_sets, keys, or stopwords", and aliases are global too), so — Q12.8
  decided 2026-10-22 — the write key's grant is narrow: `documents:*` and `collections:*` on the
  pattern, `aliases:create`, `aliases:get`, `aliases:delete` and, if SA10 uses synonym sets,
  `synonym_sets:create`, `synonym_sets:get`, `synonym_sets:delete`; plus the two read-only status
  actions `debug:list` and `metrics.json:list` (Q12.11 decided 2026-10-22: they return the server's
  version, state and memory, never a document or a name, so the version check and B8's pre-build
  memory check run with the tenant's own key); never a `list` action on a data endpoint, never
  `stats.json:list`, never `keys:*` (provisioning uses the bootstrap key). Every alias and synonym-set name
  carries the random prefix, so another tenant's names are unguessable and unlistable. `/health`
  needs no key; the old plugin displayed the key in a text field and stored it in the database;
  the harness reads its key from the environment and never writes it to results.
- **Blocking questions:** Q12.1 (decided), Q12.7 (decided), Q12.8 (decided), Q12.11 (decided)
- **Proves it:** `harness/ab/X1_keys.php` on the local server with two tenants — the search key
  cannot import, create or drop (403) and the write key can; a key scoped to one prefix cannot read
  or write the other's collections; **the alias cases (Q12.8):** (1) tenant A's write key points
  A's own alias at one of B's collections and searches through it — the server must refuse because
  the resolved collection is outside A's scope; if it returns B's documents, narrowed alias
  permissions are unsafe on a shared server and the fallback is one Typesense process per tenant;
  (2) A's key changing B's alias by name is recorded as allowed-by-the-server, with `aliases:list`
  confirmed refused, so the protection is that B's name is unguessable; (3) the same two cases for
  synonym sets; (4) the two status actions reveal no collection content: every collection and alias
  endpoint called with a key holding only `debug:list` and `metrics.json:list` is refused; a captured
  log of a full search and a full import contains no key; the setup page's HTML contains no key.
  Repeated on the shared-server proof (G6).

### X2 — Transport and network
- **Builds:** (Q12.2 decided 2026-10-22) plain HTTP accepted only for a loopback address; `https` with
  certificate verification that cannot be switched off in the plugin for any other host (the
  harness keeps its own switch); host, port and protocol validated on the setup page (admin only,
  POST with a form token) and lockable (Q9.1); keys only in the header; the `/health` probe used by
  the breaker; the checklist lines for G8: bind address, firewall to the ResourceSpace hosts, CORS
  off, a private CA on the ResourceSpace host, and the endpoint list a firewall rule needs (search,
  `multi_search`, document import and delete, collections, aliases, health, keys for provisioning).
- **Size:** S · **Depends on:** A2, G8
- **Must honour:** separate connect, read and write timeouts; the test host is localhost; a
  transport failure of any kind is a fallback, never an error shown to a user and never an empty
  result.
- **Blocking questions:** Q12.2 (decided), Q0.5 (the servers beyond the test host)
- **Proves it:** a TLS endpoint with a private certificate fails closed unless its CA is installed;
  a wrong host or port falls back; the setup page refuses an invalid host string.

### X3 — Web exposure and CLI guards
- **Builds:** (Q12.6 decided (b) 2026-10-22) a `PHP_SAPI` guard as the first lines of every script in
  `scripts/` and of the one-file `tests/` bridge (core's `batch/` and `tests/` pattern); a "Deny from
  all" `.htaccess` in `scripts/` on the `include/` precedent, with the nginx `location` rule in the
  operations page; every page action behind `checkperm('a')` and `enforcePostRequest()` with a form
  token. The harness itself lives in the planning repository and never ships; there is no `lib/`.
- **Size:** S · **Depends on:** A1
- **Must honour:** the old `reindex.php` had no guard; core ships `include/.htaccess` only; an
  offline job runs as the admin who queued it.
- **Blocking questions:** Q12.6 (decided, provisional)
- **Proves it:** an HTTP request to each script and to the bridge returns 403 or 404, or the script
  exits before including core; a GET to a setup action changes nothing.

### X4 — The security battery
- **Builds:** `harness/ab/X4_security.php`, part of the smoke set run on every change: an anonymous
  user against the `−1` placeholders; an expired user grant; a group grant after the resource leaves
  access 3; a share key limited to its collection; a user with a search filter going through
  `get_resource_access()` and `!resource`; a hidden field's value neither matched nor in
  `query_by`; an inactive field; the stale-row cases (access, state, grant, filter node, type
  changed before any sync) → the fallback, never the resource; Typesense-only mode on a gate →
  fails closed; a compiled query the server rejects → the fallback (D12). **Added 2026-10-22
  (Q12.10):** a cross-tenant read through a join (tenant A's search key filtering on a membership
  collection named with B's prefix → refused by the server); the serve-time identity guard (an alias
  pointed at a generation carrying another installation's identity → fallback and alert); the
  provisioning action (the bootstrap key in no stored option, no log line, no page after the
  action); the gate uses after I1's core change (`get_resource_access()` and the `!collection`
  checks in the request and sharing paths never reach the hook — proved in I1's battery, referenced
  here); the three share paths (an upload-share session, a request copy, `$internal_share_access`),
  each run against both engines, the upload-share case marked for a run on a real system; an
  excluded field (G9) absent from the document and the wildcard and full-text forms declining for a
  user who can see it.
- **Size:** S · **Depends on:** D6, D7, D8, D12, C8, SF2, SF4
- **Must honour:** a failing case blocks the merge regardless of the accepted-divergence list; the
  cases are written as both-engines comparisons so core's own answer is the oracle.
- **Blocking questions:** Q12.4 (decided), Q12.10 (decided), Q6.2 (decided)
- **Proves it:** green on every change; the live user matrix (H4) repeats the anonymous, expired
  and share-key cases on the test system.

### X5 — Erasure and retention
- **Builds:** the delete paths proven end to end — a hard delete removes the document, its
  memberships and its grants after the flush, after the worker and after a rebuild; a soft delete
  is hidden; an edited-away value is gone after the next flush; a deleted user's grants no longer
  match; a `--verify-absent <ref>` check on `scripts/reindex.php` for an erasure request; the
  written policy for the Typesense data directory, snapshots, the debug log and the harness results.
- **Size:** S · **Depends on:** C2, C5, B8
- **Must honour:** the queue holds refs only; P14 keeps soft-deleted documents with their state;
  `cascade_delete` is what removes memberships and grants with the resource — verified, not assumed.
  **Policy (Q12.5 decided 2026-10-22):** the index is derived data — no backups, no snapshots, rebuilt
  from the database; X7's wording is "gone from search at once, gone from disk by the next nightly
  rebuild at the latest"; Typesense's own logs off or short-rotated (agreed with infrastructure);
  plugin `debug()` lines follow core's debug log; the sync and rebuild log (G2) holds refs and
  outcomes only, thirty days by default; the harness never points at real data.
- **Blocking questions:** Q12.5 (decided)
- **Proves it:** `harness/ab/X5_erasure.php`; the policy in the operations page.

### X6 — Security review before the pilot
- **Builds:** a checklist review of the plugin's surfaces at the M2 code and again at M5: injection
  (D12), authentication and CSRF on pages, key handling, transport, error leakage (Typesense's
  messages never reach a user), the no-dependency check (Q12.9, 2026-10-22: the plugin ships no
  library and makes no outbound call except to the configured Typesense server — both confirmed by
  reading the code), log contents, file exposure, the gate uses; findings become tasks in this plan.
- **Size:** S · **Depends on:** X1–X5, G7
- **Blocking questions:** Q12.9 (decided)
- **Proves it:** the signed-off checklist in the plugin docs; no open high finding before the pilot.

### X7 — Processing description
- **Builds:** `docs/data-protection.md` in the plugin: what the index holds (generated from the
  schema register — every active field's value, confidential and custom-access resources, fields
  hidden from some users, since Q1.1 the non-indexed fields), where it lives, who can read it (key
  holders, administrators), how long, how it is deleted, the stale window, the logs, and the two
  levers an installation can pull — the exclusion list (G9) and encryption at rest for the data
  directory (G8) — written so an installation can copy it into its own records (Q12.3 decided
  2026-10-22: the copy is accepted with that protection).
- **Size:** S · **Depends on:** X5, G4, B2
- **Blocking questions:** Q12.3 (decided)
- **Proves it:** an administrator can answer "what does Typesense hold about resource N, and who
  can see it" from the page alone.
