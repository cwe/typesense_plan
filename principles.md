# Principles for the rebuild

The plugin is being written again from this plan. The existing plugin code is not the base; what was
learned from it is. These rules exist to stop fragmented development: many small changes, each
reasonable, made in different sessions with no shared record of why.

1. **The task list is the only backlog.** Nothing is built that is not a task in `tasks/`. A problem
   found while working becomes a new task (and a question, if it needs a decision), not a fix on the
   side.
2. **A task starts only when it is unblocked.** Its *blocking questions* are answered in
   `decisions.md` and the tasks it depends on are done. No guessing at a decision to keep moving.
3. **One task, one branch, one battery.** Each task is one piece of work that one session can finish,
   on its own branch, proved by the test named in the task. It merges when its battery is green and
   the accepted-divergence list is unchanged or deliberately updated.
4. **Learnings live in the plan, not in chat.** Every verified fact a task must respect is in its
   *must honour* list. A fact discovered during a task is added there (and to the plugin docs where
   users or developers need it) before the task closes.
5. **Every decision has a reason, and reversals are new rows.** `decisions.md` is append-only. A
   changed mind is a new row that references the old one.
6. **Old code is reused only on purpose.** Nothing from the previous plugin is copied by default. A
   task may say "reuse X from the old code, reviewed" when that is the better choice; the test
   harness is the obvious candidate (Q10.11).
7. **Parity is the default; differences are decided.** Where core and the plugin would differ, the
   task either reproduces core, declines to core, or records an accepted difference with a reason.
   Returning *everything* or *nothing* where core returns an answer is never an accepted difference.
8. **Code comments are terse; rationale and measurements go in the docs.**
9. **No deployment without a reindex note.** A task that changes the index says whether it needs a
   rebuild, and the deployment follows the operations page.
10. **Claude stages, Chris commits and pushes** — unless Q0.8 decides otherwise.
11. **Match the rule, not the number.** Parity means the same rule as core, proved on the designed
    harness fixture with expected sets derived by hand; a live total on the test database is a smoke
    test, never a target. Before any divergence gets code it is binned: core's *documented*
    behaviour is reproduced; core's *incidental* behaviour (a string comparison, a MySQL quirk, a
    side effect of the keyword table) is declined to core, or served with its sensible meaning and
    recorded as a difference; a *core bug* is never reproduced. The test for "incidental" is whether
    the rule would make sense on a database with different data. Declining costs nothing — the
    hook hands the search to core and the user sees core's answer — so a form earns a mechanism
    only by being common and valuable, and gets at most one; a form that needs a second layer to
    match is declined. Where core's own functions can be called (the stemmer, the stop-word list,
    the separators, `search_filter()`, the form assembly), call them rather than imitate them. A
    second, differently shaped dataset is part of acceptance so that tuning to one database shows.
12. **Generated code reads like ResourceSpace.** New code follows the repository's own standard
    (`phpcs.xml.dist`: PSR-12 with the "ResourceSpace Standard" variations and its forbidden-function
    list; `phpstan.neon.dist` at level 1; `composer dev:phpstan`) and the conventions of the core
    file nearest in purpose: snake_case functions with the plugin prefix, docblocks of core's shape
    (a one-line summary, a blank line, `@param type $name description`, `@return`), `//` comments in
    new code, the `ps_query` / `ps_value` / `ps_array` helpers rather than raw SQL, core's helpers
    and config globals rather than re-implementations. Comments stay terse (principle 8). Mandatory, not aspirational, since 2026-10-20: the plugin ships
    inside ResourceSpace 11.1, so the code goes through ResourceSpace's own `phpcs` and `phpstan` checks. Where the
    plugin must differ (a class-based pipeline, say), the file says why in one line.
