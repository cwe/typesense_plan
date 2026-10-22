# Chunk 12 — Security and data protection

Status: planning. Depends on: 0, 1, 6, 8, 9. Feeds: tasks X1–X7, D12, C8, G8.

## Scope

What the plugin adds to the attack surface and to the places confidential and personal data live: a
second, full copy of the metadata in Typesense; the network path and the API keys; the admin page,
the CLI scripts and the vendored library; user input flowing into Typesense expressions and SQL;
the logs; the stale-index window and the searches core uses as access gates; deletion and
retention; several instances on one server.

## What we know

- **What the index holds.** Per resource: every indexed field value — including fields hidden from
  some users by permission, which are excluded only at query time — option names, dates, numbers,
  node ids, `access`, `archive`, `created_by`, creation and modified dates; memberships (collection
  refs); grants (user and group refs, access, expiry). Confidential resources (access 2: 73 on the
  test system) and custom-access resources (3) are in it in full, because granted users must be
  able to find them. It holds no file contents or paths (a `file_path` sort key or `has_file` flag
  would add presence or path). Personal data in it: contributor refs (usernames too if SA9 indexes
  them), user refs in grants, names, places and EXIF-derived values in metadata.
- **Keys.** Typesense requires a key on every endpoint except `/health`; keys can be scoped by
  action (`documents:search`, `documents:*`, `collections:*`, `aliases:*`, …) and by collection
  pattern (`<prefix>_*`). The old plugin used one key for everything, stored in the plugin's config
  in the database (`set_plugin_config`) and shown in a text field on the setup page to any `a`
  user. Core keeps its own secrets (`$mysql_password`, `$scramble_key`, `$api_scramble_key`) in
  `config.php`, not the database.
- **Transport.** The old client defaulted to `http` on `127.0.0.1:8108`, sent the key in
  `X-TYPESENSE-API-KEY`, and left curl's certificate verification at its default (on); the
  harness has an insecure switch for private certificates. The test host runs Typesense on the
  same machine as RS.
- **Web exposure.** `plugins/typesense_search/scripts/reindex.php` had no CLI guard, so it was
  reachable over HTTP; core's `batch/` scripts check `php_sapi_name()`; core ships
  `include/.htaccess` and nothing under `plugins/`. The setup page is `checkperm('a')`-guarded;
  core's forms use `generateFormToken()` and `enforcePostRequest()`.
- **Input.** All of `$search` and the parameters are user input compiled into `filter_by`, `q`,
  `sort_by` and the hydrate SQL (D12 is the escaping task). The plugin's `debug()` lines log SQL,
  parameters and search strings.
- **Stale rows.** A resource whose access, state, grant, filter node or type changed is listed by a
  stale index until the sync or the rebuild — existence, title and preview, not the file, because
  the view and download pages still run core's checks. Core itself runs `!resource<ref>` as an
  access gate for users with a search filter (`get_resource_access()`), so a stale index could
  *grant* there, not merely list (Q6.2); read-time validation (C8) is the protection. Typesense-only
  mode returns nothing for a declined search: fail-closed for a gate, a denial for a user.
- **Anonymous, external and shared access.** Grant documents store `−1` in the unused user or
  group field so an anonymous searcher (userref 0) cannot match a placeholder — a leak found and
  fixed live. A collection share key (`$k`) opens exactly one collection through core's gate (SF2);
  resource-level external keys are Q6.8.
- **Deletion.** A hard delete removes the document and, by cascade, its memberships and grants
  when the `beforedeleteresourcefromdb` row is applied (the next flush, the worker within seconds,
  the rebuild nightly); a soft delete keeps the document with the deletion state (P14), hidden by
  the archive restriction; a value edited away goes with the edit; the queue holds refs only. Other
  copies: the Typesense data directory on disk (unencrypted RocksDB), any snapshot or backup of it,
  the debug log, the harness results folder (fixture data only — the harness must never point at
  real data).
- **Several instances** on one server are separated by prefix only; one admin key reads them all.
- **Supply chain.** The official client and its HTTPlug stack would be vendored (A7); core's dev
  dependencies include `roave/security-advisories`; the plugin makes no network call other than to
  Typesense.
- **Denial of service.** A search is bounded by the 4,000-byte GET/POST switch, the 100-operation
  filter guard and the 25,000-row cap; core's own search has no such caps (`!related` with a
  keyword ran past 120 s), so the plugin is no worse than core.

## Candidate work items

- Two keys — search-only, scoped by action and prefix, for the web path; admin for the CLI — stored
  as core stores its secrets, never displayed or logged (X1).
- Certificate verification always on, localhost by default, the operations checklist for binding
  and firewalls (X2, G8).
- CLI guards and web-deny rules for `scripts/`, `lib/`, `harness/` (X3).
- The security battery: placeholders, expiry, share-key scope, gate uses, stale rows,
  Typesense-only (X4).
- Erasure and retention proven and written down (X5); a review before the pilot (X6); a
  processing description for the installation's records (X7).

## Questions to answer before coding

### Q12.1 — Keys: which, where, shown how
Two keys (a search-only key scoped to `documents:search` and `multi_search` on `<prefix>_*` for
the web path; an admin key for the rebuild and the worker) or one; stored in `config.php` as core's
secrets are (recommended) or in the plugin config in the database; the setup page shows "set" or
"not set" only; a rotation procedure (create, switch, revoke). Supersedes Q9.2 — answer here.

**Decided 2026-10-22: option (a).** Two scoped keys per installation — search-only for the request path,
write for the rebuild and worker — stored as plugin options, lockable and hidden from `config.php`
(Q9.1), shown as set / not set only; a provisioning action takes the bootstrap key once, never stores
it, and creates both keys with the right scopes; rotation is the same action; keys and headers
redacted from every log line.

### Q12.2 — Network placement and transport
Typesense bound to localhost or a private network; TLS mandatory when it is not local; certificate
verification always on in the plugin (the harness's insecure switch never reaches production); the
endpoints the plugin needs (search, `multi_search`, document import and delete, collections,
aliases, synonyms, health) listed so a firewall rule can be written.

**Decided 2026-10-22: option (a).** Plain HTTP only for loopback; `https` with verification that cannot be
switched off for any other host; host, port and protocol validated and lockable; keys only in the
header; the server-side checklist and the endpoint list in G8. Local versus private network is
infrastructure's choice.

### Q12.3 — Is a full metadata copy in Typesense acceptable?
Confidential and custom-access resources, hidden-field values and personal data are in the index so
that granted users can search them. Options: accept, protect the server (Q12.1, Q12.2) and document
it (X7); exclude confidential resources or hidden fields from the index — then any search that could
include them must decline for granted users, which the plugin cannot know in advance, so those
users lose Typesense; or filesystem encryption at rest for the data directory. Recommendation:
accept with protection, and encryption at rest if the installation's policy asks for it.

**Decided 2026-10-22: accept with protection.** The copy holds every active field's value (Q1.1), guarded
by scoped keys and TLS (Q12.1, Q12.2) and described in X7; the exclusion list (G9) and encryption at
rest (G8, infrastructure's option) are the levers named there. Excluding confidential resources or
hidden fields would make granted users' searches decline wholesale.

### Q12.4 — The stale-row window and the gate uses, as a security requirement
Confirm C8 option B (fall back for the whole search when any row fails the permission re-check)
ships in M2; confirm the gate uses — `get_resource_access()`'s `!resource`, share gates, request
copies, submit-pending — bypass the index or are always validated (Q6.2); confirm Typesense-only
mode is never enabled for real users (Q0.6, Q6.5). What listing window (title and preview) is
acceptable before the hooks land: the rebuild interval today, the next page load after C3?

**Decided 2026-10-22: option (a).** The gate call sites bypass external providers in core (I1's change
list), so access decisions always come from MySQL; C8 option B ships in M2; Typesense-only never for
real users; the window is request end for the actor and within a minute for others, with the
newly-permitted-but-absent residue documented in X7.

### Q12.5 — Erasure and retention
Hard delete → removal at the flush, with the worker and the rebuild as backstops; soft delete →
retained with the deletion state until purge; edited-away values → the next flush. Agree, and decide
the policy for the Typesense data directory (no backups, or the database backups' retention),
snapshots, the debug log (it holds search strings) and the harness results.

**Decided 2026-10-22: option (a).** The index is derived: no backups or snapshots, rebuilt from the
database. Delete paths as designed (flush, worker, rebuild; soft delete hidden until purge;
edited-away values gone at the next flush); "gone from search at once, gone from disk by the next
nightly rebuild at the latest". Typesense's own logs off or short-rotated (meeting agenda); plugin
debug lines follow core's debug log; the sync log holds refs and outcomes, thirty days by default;
the harness never touches real data.

### Q12.6 — Web exposure of plugin files
`scripts/`, `lib/` and `harness/` sit under the web root. Ship `.htaccess` deny rules plus an nginx
note, CLI guards in every script, and exclude `harness/` from deployments — or keep the harness
outside the plugin directory altogether (Q10.1)?

**Decided 2026-10-22: option (b), provisional.** The harness lives outside the plugin and the release —
in the planning repository, found through a path setting, with only the one-file `tests/` bridge in
the plugin. What ships is guarded as core guards `batch/` and `tests/`: a `PHP_SAPI` check and a
deny `.htaccess` in `scripts/`, the nginx rule documented, page actions behind `checkperm('a')` and a
form token. Chris may revisit.

### Q12.7 — Logging and redaction
Never log a key or request headers; search strings and SQL in `debug()` output are personal data —
only behind the debug switch, with the retention of Q12.5; the explain tool (G7) prints compiled
parameters — admin and CLI only.

**Decided 2026-10-22: confirmed.** Keys and headers redacted everywhere (Q12.1); the sync log holds refs
and outcomes only (Q12.5); the per-search `debug()` line, search string included, only behind
`$debug_log` under core's existing policy; the explain tool admin-only, setup page and CLI, never a
search page, never a key.

### Q12.8 — Keys per instance
When several instances on one server belong to different owners, one admin key reads every prefix.
One key pair per prefix, or accept (Q9.8)? Since Q9.8 is decided as a requirement, the
recommendation is one key pair per tenant, scoped to `<prefix>_*` — with Q12.11 deciding what, if
anything, sees the whole server.

**Decided 2026-10-22: one scoped pair per tenant, aliases kept, narrow grants, unguessable names.**
The collection pattern confines collection endpoints only (docs 30.0), so the write key gets
create / get / delete on aliases (and on synonym sets if SA10 uses them) but never `list`, every name
carries the random prefix, a serve-time guard verifies the alias resolves to this installation's own
generation, and X1's battery tests the two alias cases; if the server lets a key read another
tenant's collection through its own alias, the fallback is one Typesense process per tenant. Chris:
aliases are definitely needed; dropping them was rejected.

### Q12.9 — Supply chain
The vendored client pinned, `composer audit` run in CI and on upgrade, the upgrade reviewed; no
other runtime dependency; no outbound call except Typesense. Agree?

**Decided 2026-10-22: settled by Q9.10.** No dependency, no vendored library, no outbound call except
to the configured Typesense server; X6 confirms both by reading the code. Typesense is the one
external component: who tracks its advisories is on the infrastructure meeting's agenda, and G8's
tested range moves when a new release is proved.

### Q12.10 — Anonymous, external and shared access as standing tests
The `−1` placeholder, an expired grant, a share key scoped to its collection, a user with a search
filter, a hidden field, an inactive field, the stale-row cases: run in X4's battery on every change.
Anything to add (request copies, upload-share sessions, `$internal_share_access`)?

**Decided 2026-10-22: X4 as written plus six cases** — the cross-tenant join read, the serve-time
identity guard, the provisioning action's bootstrap-key handling, the gate uses after I1's change,
the three share paths (upload-share session, request copy, `$internal_share_access`), and an excluded
field (G9). About twenty cases, a few seconds locally, every one blocking a merge.

### Q12.11 — A server-wide key for diagnostics, or tenant keys only?
Tenant-scoped keys cannot read `/debug`, `/stats.json` or list every collection, so a tenant's setup
page cannot show the server's version, limits or total load without a server-wide admin key — which
would also let that tenant read the others. Options: tenant keys only and the server check degrades
to `/health` plus the tenant's own collections; one server-wide read-only diagnostics key held by the
operator and entered per tenant; the operator runs the server checks outside RS. Also to verify
before deciding: whether a key scoped to `<prefix>_*` is also prevented from creating or reading
aliases and synonym sets of another prefix (alias and synonym-set names are not collections).

**Decided 2026-10-22: option (b).** Tenant keys only, in the sense that matters — no key that can read
another tenant's data — but the write key carries the two read-only status actions `debug:list` and
`metrics.json:list`, so the version check (Q1.9) and the pre-build memory check (B8, Q1.11) work with
the tenant's own key and the setup page can show server version and memory. No `stats.json:list`.
X1 confirms the two actions reveal no collection content. The reply's "none" becomes "none that can
read data".

## Done when

Keys, transport and web exposure are settled and built (X1–X3); the security battery is green on
every change (X4); erasure is proven and the retention policy written (X5); the review is signed off
before the pilot (X6); the processing description exists (X7).
