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
