---
name: steward
description: Default change loop for an established codebase whose current module boundaries are already known. Use for bug fixes, feature additions, refactors and cleanup that fit those boundaries, and especially when tests are weakened to make CI green, dead code or duplicate helpers accumulate, a small fix starts widening a public surface, or nobody can tell which changes need approval. Steward also detects architecture transitions: if work creates/deletes/splits/merges a module, moves capability ownership, changes dependency policy, repeatedly crosses the same boundary, or requires a neighbor's internals, invoke/load `surveyor` before continuing the structural part, then return here for implementation. Output is a change tier, integrity/ratchet checks, observable verification, and friction feedback. Do NOT use this to decide what to build (producer), or as general coding-style advice.
---

# Steward

The failure this prevents: an agent is asked to fix a date-parsing bug. It fixes the bug, and one unrelated test starts failing. It adds `@pytest.mark.skip(reason="flaky")` to that test, writes a second `parse_date` next to the first because the existing one had a caller it didn't want to disturb, and widens a module's public surface by one function so it can reach a value it needed. CI is green. The diff looks small. Nobody objects.

Do that two hundred times and the repository has a test suite that proves nothing, four date parsers, and no boundaries left. Each individual step was locally reasonable — that's what makes this the steady-state failure rather than a bad-agent failure.

`producer` decides what to build. `surveyor` shapes where things live. Steward governs work **inside the current shape**. When the shape itself becomes part of a later feature, Surveyor re-enters; repositories do not move through these skills only once.

## Step 0 — Decide whether to run at all

Skip it, say nothing about it, and just do the work when:

- There is no test suite and no check that can fail. Every mechanism here keys off an existing gate; without one there is nothing to protect. Get one test that covers the thing being changed, then continue.
- The repository is a scratchpad, a spike, or a notebook — code with no future readers.
- The user is asking a question about the code rather than changing it.
- The change is confined to one file that has no callers, such as a script or a one-off migration.

Run it on any change to code that someone will still be running next quarter.

**Budget: the process may not cost more than the change.** A tier-1 fix gets a tier check, the existing test run, and nothing else — no ceremony, no report, no friction entry. If steward turns a three-line fix into a document, it has become the entropy it exists to remove.


## Routing: default here, escalate structure

Do not choose between Steward and Surveyor based on repository age. Choose based on **what the current change alters**.

| Current change | Owner |
|---|---|
| Behavior changes inside an existing module | `steward` |
| Internal refactor with the same public/dependency shape | `steward` |
| Public contract changes but ownership stays put | `steward` tier 3 |
| New/deleted/split/merged module | `surveyor` first, then `steward` |
| Capability ownership moves between modules | `surveyor` first, then `steward` |
| Dependency policy/direction must change | `surveyor` first, then `steward` |
| A local change needs a neighbor's internals | `surveyor` question before more context is loaded |
| Repeated friction in the same module | `surveyor` triage |

This distinction is deliberate: Steward owns **change discipline**; Surveyor owns **change topology**.

If a tier-4 signal appears after Steward has already been selected, do not keep going under Steward just because the session started here. Invoke/load Surveyor and apply its boundary procedure to the structural part. If the environment cannot dispatch another skill mid-task, read and follow the installed Surveyor instructions rather than reproducing a second architecture method here. Then return to this loop for the implementation.

## The one rule that governs everything else

> When the result is hard to reach and the measurement is easy to edit, an agent will edit the measurement. Not from malice — because "make this green" and "make this correct" look identical from the inside, and one of them is reachable.

Everything below is a consequence:

- **Tiers** decide what this task is allowed to change at all.
- **Verification integrity** makes the measurement more expensive to move than the result.
- **Friction** gives the legitimate channel for "the measurement is genuinely wrong," so there has to be no illegitimate one.
- **Entropy** covers the one thing no measurement asks for and therefore never happens: deletion.

The corollary applies to this skill too: a rule here that nothing can check is a rule that will be quietly dropped in month three. Where a check exists, it is named.

## The task loop

Every unit of steady-state work has the same shape.

1. **Read down, not across.** Root `AGENTS.md`, then the nearest module `AGENTS.md`, then the module and direct dependencies' public surfaces. Stop there. If correctness requires a neighbor's internals, do not solve that by reading sideways; treat it as an architecture-transition signal for `surveyor`.
2. **Classify.** Run the tier check before writing code (`tiers.md`). The tier is a property of the diff, not of how big the change feels.
3. **Do the work in that tier only.** If the work turns out to need a higher tier, stop and re-enter at that tier. Do not finish the change and mention it afterwards.
4. **Verify.** The existing gates, plus the integrity check (`verification.md`). Green is necessary and not sufficient — done is defined by the scenario, not the exit code. Observable behavior is verified from outside the changed implementation when practical.
5. **Fresh review for tiers 2 and 3.** When an independent reviewer/subagent is available, review from a clean context using the behavior/issue, applicable `AGENTS.md`, the diff, and verification output — not the implementation conversation. If independent context is unavailable, emit that same review packet for the next reviewer; do not call self-review 'fresh'. Tier 1 skips this unless repository policy already requires review. Details in `verification.md`.
6. **Close.** Either done, or a stuck report. Both are terminal, both are acceptable. Log friction only if there was friction (`friction.md`).

## Change tiers

| Tier | Mechanical test | What it costs |
|---|---|---|
| **1 · Internal** | No test file and no public-surface file in the diff | Just do it. Existing tests are the whole gate |
| **2 · Behavior** | Test files in the diff, public surface untouched | Change the tests in their own commit, *before* the implementation, and say in one line what behavior changed |
| **3 · Contract** | A public surface, schema, migration, or API file in the diff | Stop. Other people's code depends on this. Name the callers and get a human yes |
| **4 · Structure** | Module set/ownership changes, or dependency policy changes | Architecture transition: run `surveyor`, then return here for implementation |

Two properties matter more than the boundaries themselves. The test is **mechanical** — it reads the diff, so it does not depend on the agent's estimate of how big a change is, and that estimate is always rounded down. And the tier is **discovered before the work, checked again after**: a change that starts as tier 1 and ends touching a public file was a tier-3 change all along, discovered late.

`tiers.md` has the classifier script, the config, and the cases that look ambiguous.

## The three ratchets

A steady-state repository accumulates three counters that only ever go up on their own: forbidden imports, disabled tests, dead code. Ratcheting means recording today's number and failing the build when it rises — not requiring it to be zero.

| Counter | Owner |
|---|---|
| Boundary violations | `surveyor`, `retrofit.md` |
| Skipped, xfailed and deleted assertions | `verification.md` |
| Dead code and unused dependencies | `entropy.md` |

The shared mechanism, including why the check must also fail when a count goes *down*, is in `ratchets.md`. Add at most one ratchet at a time and only for a count that is currently non-zero — a ratchet at zero is just a rule, and a rule is cheaper.

## Verification integrity

The four rules, in full. Everything else about this is in `verification.md`, including what the tampering actually looks like in each language and the check that detects it.

1. **In a tier-1 change, test files are read-only.** Not "avoid changing" — read-only. If a test must change, the change is tier 2 and the test edit is a separate commit that lands first.
2. **A test is never disabled to make a change land.** No skip, no xfail, no ignore, no rename out of the discovery pattern, no `continue-on-error`, no widened tolerance, no assertion turned into a truthiness check. A test that is genuinely wrong is deleted, in its own commit, with the reason.
3. **Green is not done.** Done is the scenario in `SPEC.md`, or the behavior described in the issue. Passing tests are evidence, not the definition.
4. **Stuck is a terminal state, and reporting it is a success.** When the change cannot be made without violating one of the above, stop and write the stuck report. Loosening the acceptance criteria to finish is the single worst available move, and it is always the most available one.

## Friction

The reverse channel: implementation is where design errors are discovered first, and by default that discovery is thrown away at the end of the session.

Append to `FRICTION.md` when — and only when — one of these actually happened: a stated rule was wrong or ambiguous, the design forced code you consider bad, or the first correct-looking approach was blocked by something no document mentioned. Five fields, three minutes. Not a report on the task.

Silence is a valid outcome for most tasks. A friction log that gets an entry every time is a form filled in, and nobody reads forms. Format, triage cadence, and what to do with the entries: `friction.md`.

## Entropy

Nothing in a normal workflow ever asks for a deletion, so deletion needs its own task type or it does not happen. Adding is local and safe; removing requires proving a global negative, which is exactly the kind of work an agent avoids under a deadline.

Read `entropy.md` before deleting anything — in particular the list of places where the caller graph lies (dependency injection, reflection, entry points declared in manifests, template strings, serialized data). A deletion that a compiler approves and a plugin registry does not is a production incident.

## Definition of done for a steady-state task

- The scenario behaves as described, checked by running it through the narrowest stable surface outside the changed implementation when practical, not by reading the diff.
- The existing gates pass, and none of them were modified in this change.
- The tier the change ended in is the tier it was done under; any tier-4 portion passed through Surveyor before implementation.
- Tier-2 and tier-3 changes received a fresh-context review when independent context was available; otherwise a review packet was produced for the next reviewer, without pretending self-review was independent.
- Anything found and not fixed is written down — in `Found · Not doing` if there is a `SPEC.md`, otherwise in the stuck report.

## Taste degradation check

Passing tests is necessary but does not prove that a change preserves the project's design judgment.

Before finishing, check whether the change introduced:

- unnecessary abstraction layers
- broader concepts than the feature requires
- hidden state or surprising behavior
- duplicated mechanisms instead of clearer ownership
- features that make the project look larger while making it harder to understand

A good change should improve capability without degrading the project's mental model.

## Self-check before you finish

- Did any test file appear in a diff that I classified as tier 1?
- Did I add a skip, an xfail, a widened tolerance, or a mock that returns the value the assertion is looking for?
- Did I add code that duplicates something already in the repository because touching the original felt risky?
- Did I discover a contract change halfway through and keep going instead of stopping?
- Did I discover a structural change and continue under Steward instead of re-entering Surveyor?
- Did I need a neighbor's internals and treat that as 'more context' rather than an architecture signal?
- Did I widen a module's public surface to reach one value? (That is tier 3, no matter how small the addition looks.)
- Am I reporting "done" for something I have not observed working?
- Did I run into a rule that was wrong and fail to write it down?
