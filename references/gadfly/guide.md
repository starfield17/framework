# Gadfly

The failure this prevents comes in two versions.

**The unchosen path.** An agent is asked for a spreadsheet that renders a base64 stream as an animated QR code, formulas only. It does what the request literally describes: it writes formula strings, sheet by sheet. Every piece is locally correct. The result is tens of thousands of hand-assembled formulas whose correctness can only be observed in the final pixels, and a character-counting bug that no test is positioned to see. Nobody chose this path. It was simply the first one to appear, and nothing ever competed with it.

**The evaporated path.** A better path *was* chosen — "compile the formulas from a small IR" — and written into the prompt. The agent built the IR, used it for the first sheet, and then went back to string concatenation for the rest, because at every individual step the string was the shorter move. The tests passed. The chosen architecture existed only as a sentence in a conversation.

Gadfly addresses both: it makes the default path compete before commitment, and it turns the winner into a check that cannot be silently dropped.

`producer` decides what to build and why. Gadfly decides **what kind of problem this is** — the solution path. `surveyor` decides where things live; `steward` governs changes inside the settled shape. Gadfly has no fixed place in that sequence: it runs at the start when the path is open, and it re-enters whenever the path itself turns out to be the cause of the trouble.

**Not the same friction as `FRICTION.md`.** `FRICTION.md` records friction *received* — reality pushed back on a rule during implementation. Gadfly *manufactures* friction deliberately, before the code exists, against a path nobody has tested yet. One is evidence and the other is hypothesis. Received friction is gadfly's best input: a repeated entry is often a frame failing in slow motion.

## Step 0 — Decide whether to run at all

Skip it, say nothing about it, and just do the work when:

- A standard solution exists and nothing about this case makes it doubtful — a CRUD endpoint, a well-known file format with a mature library.
- Failure is cheap: redoing the work costs less than this discussion. A producer T1 throwaway never needs a gadfly.
- The user has fixed the approach and wants it executed, not examined. (If they ask "is this the right way?", that is an invitation — run.)
- The work is ordinary steward work inside a path that is not in question.

Run it when any of these holds:

- **Stakes** — the user says, or the task makes plain, that it is easy to fail or expensive to redo.
- **Medium mismatch** — the problem is being solved somewhere not built for it. A default imported from the "normal" medium will usually fit badly.
- **Instance smell** — the plan contains dozens of near-identical hand-written pieces. That is usually a sign the task is one instance of a class that should be generated.
- **Stuck** — a steward stuck report, or two attempts that failed for *different* reasons.
- **Repeat friction** — the same constraint in `FRICTION.md` twice, and neither changing the rule nor changing the code is defensible.
- **A hunch** — the user offers an analogy or a half-formed direction. This is the most productive trigger; see the section on hunches.

**Budget: one round, at most three framings including the default, output of about twenty lines.** A gadfly that produces an essay has turned deliberation into procrastination. If one round does not converge, the answer is a probe (terminal state B), not a second round of argument.

## The one rule that governs everything else

> A solution path counts as an alternative only if it moves where the difficulty lives.

Cosmetic variants — another library, another language, another folder layout — leave the hard part exactly where it was. Generating them is the counterfeit version of this skill: it looks like deliberation and changes nothing. A real alternative relocates the hard part. Examples: from "forty thousand formulas correct at once" to "a dozen operator translations, each checkable alone"; from "compute the layout at runtime" to "a constant table produced once"; from "keep the state consistent" to "recompute it from an append-only log".

The test is mechanical. For each framing, fill in two blanks:

```
the hard part is:     ______
correctness is checked at: ______
```

If two framings produce the same two answers, they are the same framing; drop one.

Corollary: **the default gets written down as a framing too.** It is the path nobody chose. Naming it is what makes it possible for it to lose — and just as possible for it to win on the merits.

## The loop

1. **Name the default.** One line: the path that would happen with no intervention, plus its two blanks. Do this first and honestly. A model's first idea is usually the consensus idea, which is exactly why it must be on paper before anything else.
2. **List its load-bearing assumptions.** Three to five things the default silently takes as given: "formulas are written, not generated"; "the spreadsheet is a calculator"; "the whole string exists in one place at some point". An assumption is load-bearing if negating it changes one of the two blanks.
3. **Generate rivals by breaking assumptions, not by brainstorming.** Each rival negates one load-bearing assumption and follows the consequence all the way through. The moves that reliably produce real rivals are in `drills.md`: lift to the class, change the representation, move where correctness lives, freeze what varies, start from the oracle. At most two rivals. **Write every rival before judging any of them** — evaluating while generating collapses back to the default within a sentence.
4. **Find each framing's breakpoint.** Every framing fails somewhere, and an analogy always does. Name where. Then write the line that matters most: *what the break forces*. "Like Java → JVM" breaks because a spreadsheet has no loops and no mutable state; that forces the intermediate layer to be static dataflow, with bounded loops unrolled into rows. A breakpoint that forces nothing is just a caveat. One that forces a shape is often the most valuable line in the output.
5. **Decide, or probe.** See the terminal states below.
6. **Pin the winner.** Turn it into an invariant, an oracle, and prohibitions, as described under "Pinning". An unpinned frame is the second failure above, waiting to happen.

## When the user brings a hunch

A user's half-formed idea — "what if it's like a compiler?" — is the single best input this skill gets. It is also the easiest to mishandle, in one of three ways: applauding it, quietly replacing it with the default, or burying it under a list of alternatives.

Instead, run steps 4 and 6 on the hunch directly: where does it break, what does the break force, and what check would keep it alive through implementation. Only then compare it against the default.

Hunches are usually right about the *level* — what kind of thing the solution is — and wrong in the details. The breakpoint drill keeps the level and repairs the details. If the hunch fails the one-rule test and moves nothing, say so in one sentence and move on. Flattering a hunch wastes it.

## Terminal states

**A · The default survives.** The rivals were generated honestly and lost. Record why in one line. This is a success, not a waste: a default that has survived an argument is no longer a default, it is a decision. Guard against the opposite bias as well — novelty is not a merit, and a gadfly that always overturns the default is as uninformative as one that never does.

**B · Probe.** Two framings are live and argument cannot separate them. Give each a **kill probe**: the cheapest experiment — one file, under about half an hour — whose result would eliminate it. "Does Excel's `MID` split a surrogate pair?" "Does a 5,000-cell LFSR grid recalculate in under 200 ms?" Run the probes, then decide. A probe that cannot eliminate anything is a prototype; do not call it a probe.

**C · A rival wins.** Adopt it, then pin it.

## Pinning: a frame that nothing checks is a wish

Implementation pressure always pulls toward the default, because the default is the locally shortest move at every single step. The chosen frame therefore has to leave behind three things, each checkable without asking its author.

- **A frame invariant and its check.** A statement a third party could verify from the code alone, plus a command that fails when it is violated. Examples: "every formula is emitted by `ir/` — check: no string literal beginning with `=` outside `ir/`"; "every IR operation has an interpreter with target semantics — check: a test that enumerates the operations". If the invariant is about where code may live, it is a boundary rule: hand it to `surveyor` and enforce it with the boundary check. Otherwise it becomes a test in steward's gates. As with every check in this framework, make it fail once on purpose before trusting it.
- **The frame's oracle.** A frame usually implies its own verification layer. A compiler frame implies differential testing: reference implementation, IR interpreter, and real engine must agree. Name the oracle, and state **whose semantics it uses**. It must use the target's semantics, not the authoring language's; a reference that counts characters the way Python does will certify a spreadsheet that counts them differently. The oracle becomes part of steward's definition of done.
- **Prohibitions.** If a `SPEC.md` exists, add numbered do-not-build entries derived from the frame, such as "N4 No hand-written formula strings outside `ir/`".

If none of these three can be written, the frame is too vague to adopt. Go back to B and probe.

## The artifact

For a new project, a `## Frame` section in `SPEC.md`, directly under `## Do not build` — it gets reread nearly as often. For an existing repository, the block goes in the stuck report or the change description. Either way, the prose is scaffolding. **The check is the artifact.** Once the invariant is enforced, the paragraph that argued for it may be deleted without loss.

```markdown
## Frame
Default:     <one line> · hard part: <…> · checked at: <…>
Rival 1:     <one line> · breaks: <assumption> · hard part: <…> · checked at: <…>
Rival 2:     <optional, same shape>
Breakpoints: <framing> breaks at <…> → forces <…>
Decision:    A | B | C — <one line why>
Probes:      (B only) <experiment> — eliminates <framing> if <result>
Invariant:   <statement> — check: <command>
Oracle:      <what verifies correctness, and whose semantics it uses>
```

`drills.md` holds the five moves in detail, the ways gadfly itself goes wrong, and a fully worked case.

## Self-check before you finish

- Did I write down the default, with its two blanks, before generating anything else?
- Does every rival change *the hard part* or *checked at*? Or is one of them a library swap?
- Did I judge any framing before all of them were written?
- For every analogy, did I name where it breaks and what the break forces?
- If the user brought a hunch, did I work *on* it — or did I substitute my own default?
- Did I prefer a rival because it is new? Would I still pick it if it had been the default?
- Does the winner have an invariant with a command that fails, and have I watched it fail?
- Does the oracle use the target system's semantics, or my language's?
- Did this cost more than the task it serves? If so, Step 0 said skip.
