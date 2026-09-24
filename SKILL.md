---
name: framework
description: "Route software work through four focused modes: clarify the goal of an uncertain new project, challenge a doubtful solution path, establish or change repository boundaries, and protect correctness during ongoing changes. Use for new projects whose intent needs shaping or meaningful changes to maintained code. Skip code-reading questions and trivial one-off work."
---

# Framework

Use one entry point for the life of a project. Choose the next mode from the work in front of you; do not run all four as a checklist. Read only the guide needed for that mode and its relevant supporting files.

## Route

1. **Goal unclear for a new project?** Read [Producer](references/producer/guide.md). It either recommends an existing solution, defines a small spike, or writes a short `SPEC.md`. Re-enter Producer for an established project only when its goal or scope is explicitly reopened.
2. **Solution path in doubt?** Read [Gadfly](references/gadfly/guide.md) before committing to the path. An adopted Frame supplies an invariant and an oracle. Gadfly may also re-enter after implementation gets stuck or repeated friction points to the approach.
3. **Ownership or dependency shape changing?** Read [Surveyor](references/surveyor/guide.md) before the structural part. It establishes the repository map and an executable boundary check, then hands implementation to Steward. Ordinary behavior inside an existing owner does not need this stage.
4. **Changing maintained code within a settled shape?** Read [Steward](references/steward/guide.md). It classifies the change, protects verification, and checks observable behavior. Small throwaway work and read-only questions can skip this stage.

When more than one condition holds, resolve the goal first, then the solution path, then structural boundaries, then implementation. A stage that does not apply produces no artifact.

## Re-entry and handoffs

- A Gadfly invariant about code location or dependencies goes into Surveyor's boundary check. Other invariants and the oracle become part of Steward's verification.
- A Steward change that creates, removes, splits, or merges modules, moves capability ownership, or changes dependency policy enters Surveyor before that structural work and returns to Steward afterward.
- Repeated friction in the same module goes to Surveyor. Repeated friction with the same constraint that neither a rule change nor a code change resolves goes to Gadfly. A genuine reset of product intent or scope goes to Producer.
- Preserve the existing artifacts: `SPEC.md` for intent, `AGENTS.md` and a failing boundary command for structure, a Frame invariant and check for a contested path, and `FRICTION.md` only for friction actually encountered. Do not create an artifact for a skipped stage.
