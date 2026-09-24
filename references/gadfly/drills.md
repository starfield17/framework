# Drills

Read this when you are at step 3 or step 4 of the gadfly loop. Each move is a way to break one load-bearing assumption of the default in a direction that actually moves the difficulty. You will rarely need more than two moves; pick the ones whose question has an uncomfortable answer.

Contents: the five moves · the breakpoint drill · how gadfly itself goes wrong · worked case.

## The five moves

### 1 · Lift to the class

**Question:** what is this an instance of — and is solving the class cheaper than solving the instance?

A more general problem is sometimes the easier one, because the general version has structure that the instance hides. For example, "write these forty thousand formulas" is an instance of "translate a program into formulas". The class version has a dozen operations to get right; the instance version has forty thousand strings.

**Smell that calls for it:** the plan repeats a near-identical piece dozens of times.

**How it goes wrong:** lifting for *future* instances. That is the speculative abstraction `producer` and `surveyor` exist to forbid. The line between the two is concrete: lift when **this instance already contains the repetition**, not when a second instance might appear someday. A generator for forty thousand formulas is justified today. A plugin system for a second export format that nobody has asked for is not.

### 2 · Change the representation

**Question:** what else could this medium, or this data, be seen as?

A spreadsheet can be seen as a calculator, or as a static dataflow circuit. Git can be seen as a version-control tool, or as a content-addressed graph. A queue can be seen as a mechanism, or as a log. What counts as a small step depends entirely on the representation: a solution that is far away in one view can be adjacent in another.

**How it goes wrong:** a metaphor with no consequence — "think of it as an ecosystem". Apply the one-rule test. If the new view does not change the hard part or where correctness is checked, it is decoration.

### 3 · Move where correctness lives

**Question:** where would a bug be caught today — and could it be caught somewhere smaller, earlier, and more checkable?

"Caught by looking at the final QR code" is large, late, and opaque. "Caught when one operator's translation disagrees with its interpreter" is small, early, and precise. Architecture is largely the decision about where errors can occur and where they are caught. This move makes that decision explicit.

**How it goes wrong:** moving the check somewhere that is easier to satisfy but no longer measures the goal. That is steward's rule — the measurement is easier to edit than the result — applied at the design stage.

### 4 · Freeze what varies

**Question:** which variables can be fixed by decision without breaking the goal?

Fixing the QR version, error-correction level, and mask turns the whole module layout into a constant table. It removes a runtime computation instead of implementing one. Many clean designs are really just a decision to stop supporting a degree of freedom nobody needed.

**How it goes wrong:** freezing something the goal depends on — for instance, fixing the input length when the requirement is "arbitrary length". Check every freeze against `Done when` in `SPEC.md`, or against the stated task.

### 5 · Start from the oracle

**Question:** how will we know it is right — and what path does that answer imply?

Work backward from verification. If the only credible oracle is "a real decoder scans every frame and the concatenation equals the reference", the architecture must expose each frame deterministically — hence a manual frame-override input, rather than frames driven only by the clock. Paths that cannot be verified by any affordable oracle are eliminated before any comparison of their elegance.

**How it goes wrong:** choosing an oracle that is convenient rather than faithful. The typical case is checking against a reference written in the authoring language, which silently inherits that language's semantics.

## The breakpoint drill

Use this for every analogy, and for the user's hunch in particular.

1. Write the analogy as "X is like Y".
2. List the three properties of Y that the plan actually leans on. For "a spreadsheet is like the JVM", those are: it runs loops, it holds mutable state, and it executes a general instruction set.
3. Check each property in X. The first one that fails is the breakpoint. Here, a spreadsheet has no loops: each cell is computed once.
4. Write down what the break **forces**. Loops must have static bounds and be unrolled into rows; the intermediate layer must consist of dataflow operations (map, scan, gather, reduce), not instructions. The analogy was right about the level (a translation layer) and wrong about the target (a circuit rather than a CPU). A better analogy is high-level synthesis to hardware.

If step 4 produces nothing, the breakpoint is just a caveat, and the analogy is either fully sound or irrelevant. Both are worth one line.

## How gadfly itself goes wrong

These are its own failure modes, most of them failures of an agent's defaults.

| Failure | What it looks like | What to do instead |
|---|---|---|
| Cosmetic rivals | "Option B: use openpyxl instead of xlsxwriter" | Apply the two-blank test; drop rivals that do not move the difficulty |
| Premature convergence | The second rival arrives already rebutted | Write all rivals first; judge afterwards |
| Performative contrarianism | Every default gets attacked, whatever its merits | Terminal state A is a success; say so plainly |
| Novelty bias | The rival wins because it is interesting | Ask: would I pick it if it had been the default? |
| Essay mode | Three paragraphs per framing | Use the Frame block; one line per field |
| Flattering the hunch | "Great idea — here's how to do it" | Run the breakpoint drill on it first |
| Replacing the hunch | The user's idea disappears under the agent's plan | Work on the hunch before comparing it to anything |
| Evaporation | The frame is adopted and never enforced | Pin it: invariant, check, oracle — or do not adopt it |

## Worked case: a QR animation in spreadsheet formulas

Task: a `.xlsx` workbook, formulas compatible with Excel 2019 or earlier, that takes a string of up to thousands of characters, base64-encodes it, and shows one QR frame per second on F9, looping; the concatenated frames must equal the base64 string. The user notes that the task is easy to fail.

Triggers: stakes, medium mismatch, and instance smell — all three.

```markdown
## Frame
Default:     hand-write formula sheets for UTF-8, base64, RS, layout
             · hard part: ~40k formulas right at once · checked at: final pixels
Rival 1:     (user's hunch; lift to the class) Python spec → IR → formulas
             · breaks: "formulas are written, not generated"
             · hard part: ~a dozen op lowerings · checked at: per op, differential
Rival 2:     (freeze what varies) fix version 8-L, mask 0, 180 chars/frame
             · breaks: "the QR encoder must choose parameters"
             · hard part: moves from runtime layout to a generated constant table
             · checked at: the table, once, against a reference encoder
Breakpoints: "like the JVM" breaks at "no loops, no state"
             → forces a static-dataflow IR: map / scan / gather / reduce, bounded unroll
             Rival 2 breaks at "frame capacity is fixed" → forces padding of the last frame,
             which needs its own test
Decision:    C — Rivals 1 and 2 relocate different difficulties, so they compose
Invariant:   every formula is emitted by ir/ — check: no "=" string literal outside ir/
             every IR op has an interpreter — check: test enumerating ops
Oracle:      reference encoder + IR interpreter + a real engine, agreeing;
             string ops use Excel semantics (UTF-16 code units), not Python's;
             every frame driven by a manual frame override, decoded by a real QR decoder
```

What actually happened when this was built *without* pinning: the IR was implemented and then used for one of the nine formula-bearing sheets; the remaining formulas went back to string concatenation. The validator ran on LibreOffice, whose `MID` treats a non-BMP character as a single unit, so an emoji test passed, while Excel counts UTF-16 code units and is expected to split it — a gap no test was positioned to see. Per run, only the frame the clock happened to land on was checked against the reference. All three defects fall exactly where the invariant and the oracle lines above would have put a check. In short: the frame was right, and it evaporated.
