# Surveyor

The failure this prevents: an agent is asked to add one field to the export format. It reads forty files, edits nine, breaks two tests in a module it never opened, and adds a date helper to `utils/` that duplicates one already sitting in `common/`.

None of that is a reasoning failure. The repository made the wrong change easy to make and the right one hard to find. Surveyor changes the repository, not the agent.

## Step 0 — Decide whether to run at all

Skip it, say nothing about it, and just do the work when:

- The project is under roughly ten source files. Structure costs more than it saves.
- The user asked for a specific feature or fix **and it fits the existing boundaries**. Ordinary implementation belongs to `steward`; boundary work is not a prerequisite for typing.
- There is no repository yet and no code — the question is what to build. That is `producer`.
- Boundaries already exist and a check already enforces them, and the complaint is about something else.

Run it when a change that should be local isn't, or when the change itself alters the structure that future work will inherit. Mature repositories re-enter Surveyor whenever architecture transitions; they do not graduate from it permanently.

## Architecture-transition triggers

A feature can be ordinary work and still contain a structural event. Run Surveyor **before implementing the structural part** when any of these is true:

- a module is created, deleted, split, or merged;
- responsibility for a capability moves from one module to another;
- an allowed dependency edge or dependency direction must change;
- a new `shared/`, `common/`, `utils/`, service registry, plugin layer, or cross-cutting abstraction is proposed because two modules need the same thing;
- the same module appears repeatedly in `FRICTION.md`, especially for different constraints;
- a supposedly local change requires reading or editing a neighbor's internals to be correct.

Do **not** run Surveyor merely because a feature is new. A new behavior inside an existing owner is `steward`. The trigger is a change to ownership or dependency shape, not novelty.

The normal lifecycle is therefore not `surveyor → steward` once. It is:

```text
surveyor → steward → steward → surveyor → steward → ...
             local work       architecture transition
```

`gadfly` can enter at any point in that line. It runs when the solution path — not the boundaries — is what is in question. It hands back a frame invariant. If that invariant is about where code may live ("every formula is emitted from `ir/`"), it is a boundary rule, and it gets enforced here like any other: in the dependency policy, by the one command, after watching it fail once.

**Budget: one new level of structure and one new command.** If the plan involves moving most of the files, it is the wrong plan — read `retrofit.md`.

## The one rule that governs everything else

Every sentence you are tempted to write into a repository instruction file is a fact, a rule, or an opinion.

| | What it is | Where it goes |
|---|---|---|
| **Fact** | True here, false elsewhere. "Auth lives in `modules/identity`." "Checks run with `make check`." | The map |
| **Rule** | Must never happen, and something breaks when it does. "`training` must not import `inference` internals." | A check. Prose only as a comment on the check |
| **Opinion** | True everywhere. "Prefer composition over inheritance." "Don't over-abstract." | Nowhere. Delete it |

Two tests decide which one you're holding:

- **Could this be false in a different repository?** If no, it's an opinion.
- **If someone violates it, does a command fail?** If no, it's an opinion in the imperative mood.

Deleting opinions is not tidiness. A capable model already holds them, so they carry no information — and they sit next to the facts, which the model *cannot* recover on its own, competing for the same attention. A repository manual that opens with three paragraphs of sound general advice has taught the agent that the file is skippable, and the agent is right.

This rule is also why this skill contains almost no design advice. Apply it to anything you are about to add here, too.

## Where boundaries go

The common mistake is cutting by **kind of code** instead of by **reason to change**. `controllers/ services/ repositories/ models/` is a filing system, not a set of boundaries: nearly every feature touches all four, so the agent must load all four, and the boundary protects nothing while costing four directories of navigation.

Cut where change stops propagating. Two tests for a proposed boundary:

> **The three changes test.** Name three changes plausibly coming in the next few months. If two or more cross the boundary, it is in the wrong place.

> **The fresh-context test.** Can a fresh agent make a normal change inside this boundary after reading only: (1) root `AGENTS.md`, (2) the nearest module `AGENTS.md`, (3) this module, and (4) the public surfaces of its direct dependencies? If correctness requires conversation history or a neighbor's internals, the boundary is leaking, misplaced, or missing a fact.

With git history the first test is measurable instead of guessed — files that change in the same commits belong on the same side. `retrofit.md` has a script.

A boundary is not a folder name. It consists of:

- a directory owning one capability;
- a public surface — one file or package that names everything exported;
- an internal region nothing outside may import;
- its tests, adjacent;
- a declared list of what it may depend on.

Only the last two make it real. The first three are naming conventions until something enforces them.

## Concept boundaries

Architecture boundaries are not only about code ownership. They also protect the identity of a system.

A change can be technically clean and still be wrong if it expands the concept beyond what the project promises.

Examples:

- A local video compressor becoming a cloud video platform
- A remote execution primitive becoming a full remote desktop product
- A focused library becoming a framework without a concrete need

When evaluating a structural change, ask two questions:

1. Does this belong to the existing concept, or does it create a different product?
2. Does this preserve the user's existing mental model?

A boundary violation is not only "module A imports module B". It can also be "the project no longer means the same thing."

## How much structure

You must be able to name the event that pushed you up a level. "It might grow" is not an event.

| Level | Shape | Event that justifies moving up |
|---|---|---|
| **L0** | flat `src/` + `tests/` | One file does two jobs that change on different schedules |
| **L1** | `modules/` by capability, public surface each | Two capabilities with different dependencies or lifecycles |
| **L2** | ports at the edges | A second implementation exists now, or the dependency is external, unstable, or hard to test |
| **L3** | domain model inside one module | Real invariants, state transitions, policies — not CRUD |
| **L4** | separate deployables | Independent release, scaling, or security boundary is actually required |

Never skip a level in anticipation. Moving up late costs one refactor; moving up early costs a tax on every change forever, and the refactor anyway when the guess turns out wrong.

L3 and L4 apply to *one module at a time*. A repository is usually L1 with one module at L2 and nothing at L3.

## The artifact

Produce four things, and no more:

1. **Root `AGENTS.md`** — the map. Facts only, one screen. Template in `templates/ROOT_AGENTS.md`.
2. **Module `AGENTS.md`** — only for modules a newcomer would get wrong. Template in `templates/MODULE_AGENTS.md`.
3. **One command** that fails on a boundary violation.
4. **The dependency policy**, in whatever machine-readable form the language offers.

Order both instruction files by **reread frequency**, not by narrative. Commands and hard rules at the top — they get read every session. Rationale and history at the bottom, or nowhere.

**Definition of done:** write a forbidden import on purpose, run the command, watch it fail, then delete the import. A check that has never failed is not known to work — a misconfigured path glob silently passes everything, and that is the most common outcome of adding architecture tests.

## Enforcement

Prose enforces nothing. Pick the cheapest mechanism that fails the build:

1. **The language.** Go's `internal/`, Rust's crate graph and `pub(crate)`, package-private visibility. Free, zero config, cannot be bypassed or misconfigured.
2. **Build or lint config.** import-linter, dependency-cruiser, depguard.
3. **A test that inspects the source.** When nothing above fits.
4. Prose. Not an option.

A frame invariant from `gadfly` is enforced the same way. It must not become a second, parallel mechanism: if it can be expressed as a forbidden import or a forbidden location, it belongs in the existing boundary check.

Prefer 1 over 2 harder than feels natural. A large share of the boundaries people reach for architecture tests to protect could just be a directory the language already refuses to let you cross.

Working configurations, one file per language — read only the one you need:

- `enforcement/python.md` — import-linter contracts
- `enforcement/typescript.md` — dependency-cruiser, package exports
- `enforcement/go.md` — `internal/`, depguard
- `enforcement/rust.md` — workspace crates
- `enforcement/fallback.md` — a ~30-line import-walking test for anything else

## Existing repositories

Do not rewrite. Read `retrofit.md`. The short version: declare the boundary you want, count the violations that exist today, and fail the build only when the count goes **up**. That converts a six-month migration into something that starts protecting you this afternoon, and it lets an agent do the cleanup incrementally without a flag day.

## Self-check before you finish

- Did I write anything that would be equally true in someone else's repository? Delete it.
- Is there a rule stated in prose that no command checks? Either make it executable or drop it.
- Did I make the check fail on purpose and watch it fail?
- Did I add structure without being able to name the event that required it?
- Could a fresh agent correctly change one module after reading only root/module instructions, that module, and its direct dependencies' public contracts?
- Did understanding the change require a neighbor's internals or prior conversation history? If yes, did I treat that as a boundary/fact failure instead of simply loading more context?
- Does every directory have an owner and a reason to change, or did a `shared/` appear because two things looked alike?
- Is root `AGENTS.md` longer than one screen?
