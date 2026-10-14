# Decisions log

Append one row per answered question. Keep the question ID so the chunk file can be updated to
match. Reasons matter more than the decision: they are what a later session needs.

| Date | Question | Decision | Reason / notes |
|---|---|---|---|
| 2026-10-13 | Q0.1 | **Fresh start for the code.** The plugin is written again from this plan; the existing plugin changes are not the base. What was learned (facts, measurements, design) carries forward through each task's *must honour* list. | Chris: "a fresh start for the code … using everything learned so far as a base … to stop fragmented development". Whether the pipeline *design* is kept is Q0.13; whether the test harness is reused is Q10.11. |

| 2026-10-14 | principle 11 (bears on Q0.2, Q0.3) | **Match the rule, not the number.** Parity is defined as core's rule, proved on the designed fixture; live totals are a smoke test. Divergences are binned as documented behaviour (reproduce), incidental behaviour (decline, or serve the sensible meaning and record the difference) or core bug (never reproduce). One mechanism per form; a form needing a second layer is declined. | Chris: "avoid over tuning (focusing on matching results exactly by adding in layers of complexity) for the test database … while still wanting to match core where possible". Decline is the zero-complexity way to match core. Q0.2 and Q0.3 stay open for the goal statement and the list of always/never acceptable approximations. |
| 2026-10-14 | principle 12 | **Generated code follows ResourceSpace's own code and commenting style** where possible: the repository's coding standard, its docblock form, naming, database helpers; comments terse. | Chris: "match ResourceSpace code + commenting style in any future generated code, where possible". Extends P36. |

Confirmed or revisited prior decisions go here too, as `P<n>` rows.
