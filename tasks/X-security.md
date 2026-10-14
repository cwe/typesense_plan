# X — Security and data protection

Seven tasks from [chunk 12](../chunks/12-security-and-data-protection.md). X1–X3 are milestone M0
(nothing talks to a server before keys, transport and web exposure are right), X4 is M2 (it runs
with the first served search and on every change after), X5 is M3, X6 is the review before the
pilot (M5), X7 is M7 with the documentation. The letter is X because J and K are catalogue groups.

A failing case in X4 blocks a merge whatever the accepted-divergence list says: a security
difference is never an accepted difference.

### X1 — Key handling
- **Builds:** two config options read from `config.php` like core's own secrets
  (`$typesense_search_search_key`, `$typesense_search_admin_key`); the client (A2) picking the key
  by operation; the setup page showing "set" / "not set" and never the value; redaction of the key
  and of request headers in every `debug()` line; the rotation procedure in the operations page
  (create the new key, switch the config, revoke the old).
- **Size:** S · **Depends on:** A1, A2
- **Must honour:** a Typesense key is scoped by actions and by collection pattern — the search key
  gets `documents:search` on `<prefix>_*` and nothing else, and both keys are per installation
  (Q12.8, A8); whether alias and synonym-set calls are also confined by the pattern is verified
  here, not assumed (Q12.11); `/health` needs no key; the old plugin
  displayed the key in a text field and stored it in the database; the harness reads its key from
  the environment and never writes it to results.
- **Blocking questions:** Q12.1, Q12.7, Q12.8, Q12.11
- **Proves it:** `harness/ab/X1_keys.php` — the search key cannot import, create or drop (the local
  server answers 403) and the admin key can; a key scoped to one prefix cannot read, write, alias
  or add synonyms for the other; a captured log of a full search and a full import
  contains no key; the setup page's HTML contains no key.

### X2 — Transport and network
- **Builds:** `https` support with certificate verification that cannot be switched off in the
  plugin (the harness keeps its own switch); host, port and protocol validated on the setup page
  (admin only, POST with a form token); the `/health` probe used by the breaker; the checklist
  lines for G8: bind address, firewall, the TLS certificate.
- **Size:** S · **Depends on:** A2, G8
- **Must honour:** separate connect, read and write timeouts; the test host is localhost; a
  transport failure of any kind is a fallback, never an error shown to a user and never an empty
  result.
- **Blocking questions:** Q12.2, Q0.5
- **Proves it:** a TLS endpoint with a private certificate fails closed unless its CA is installed;
  a wrong host or port falls back; the setup page refuses an invalid host string.

### X3 — Web exposure and CLI guards
- **Builds:** a `PHP_SAPI` guard at the top of every script and every harness entry (core's
  `batch/` pattern); `.htaccess` deny rules in `scripts/`, `lib/` and `harness/` plus an nginx
  note in the operations page; `harness/` excluded from deployments or kept CLI-only; every page
  action behind `checkperm('a')` and `enforcePostRequest()` with a form token.
- **Size:** S · **Depends on:** A1, A7
- **Must honour:** the old `reindex.php` had no guard; core ships `include/.htaccess` only; an
  offline job runs as the admin who queued it.
- **Blocking questions:** Q12.6
- **Proves it:** an HTTP request to each script returns 403 or 404, or the script exits before
  including core; a GET to a setup action changes nothing; `lib/` serves no file.

### X4 — The security battery
- **Builds:** `harness/ab/X4_security.php`, part of the smoke set run on every change: an anonymous
  user against the `−1` placeholders; an expired user grant; a group grant after the resource leaves
  access 3; a share key limited to its collection; a user with a search filter going through
  `get_resource_access()` and `!resource`; a hidden field's value neither matched nor in
  `query_by`; an inactive field; the stale-row cases (access, state, grant, filter node, type
  changed before any sync) → the fallback, never the resource; Typesense-only mode on a gate →
  fails closed; a compiled query the server rejects → the fallback (D12).
- **Size:** S · **Depends on:** D6, D7, D8, D12, C8, SF2, SF4
- **Must honour:** a failing case blocks the merge regardless of the accepted-divergence list; the
  cases are written as both-engines comparisons so core's own answer is the oracle.
- **Blocking questions:** Q12.4, Q12.10, Q6.2
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
- **Blocking questions:** Q12.5
- **Proves it:** `harness/ab/X5_erasure.php`; the policy in the operations page.

### X6 — Security review before the pilot
- **Builds:** a checklist review of the plugin's surfaces at the M2 code and again at M5: injection
  (D12), authentication and CSRF on pages, key handling, transport, error leakage (Typesense's
  messages never reach a user), dependency audit (`composer audit`), log contents, file exposure,
  the gate uses; findings become tasks in this plan.
- **Size:** S · **Depends on:** X1–X5, G7
- **Blocking questions:** Q12.9
- **Proves it:** the signed-off checklist in the plugin docs; no open high finding before the pilot.

### X7 — Processing description
- **Builds:** `docs/data-protection.md` in the plugin: what the index holds (generated from the
  schema register), where it lives, who can read it (key holders, administrators), how long, how it
  is deleted, the stale window, the logs — written so an installation can copy it into its own
  records.
- **Size:** S · **Depends on:** X5, G4, B2
- **Blocking questions:** Q12.3
- **Proves it:** an administrator can answer "what does Typesense hold about resource N, and who
  can see it" from the page alone.
