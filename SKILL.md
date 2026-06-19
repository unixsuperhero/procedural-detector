---
name: procedural-detector
description: Scan a codebase for code smells that come from a procedural mindset — verb_noun function names, temp-variable shuffling, primitive obsession, grab-bag hash returns, multi-purpose functions, and call-site-driven signatures. Produces a findings report with a concrete OO/encapsulated refactor sketch per finding. Use when the user wants to audit for procedural smells, find primitive obsession, detect missing encapsulation, review code for OO design quality, or says "find procedural smells", "detect code smells", "run procedural-detector".
---

# Procedural Detector

Audits a codebase for smells that signal a **procedural mentality** — logic written as
free functions shuffling raw data, instead of behavior encapsulated in objects that own
their data. Language-agnostic: judge by the principle, not by syntax.

Default mode: **whole-codebase sweep → report + refactor sketch.** No edits are made.
If the user names a path/file/layer, scope to that instead.

## Workflow

1. **Scope.** Identify source files to scan (skip vendored deps, generated code, build
   output, tests unless asked). For large repos, sweep in batches by directory.
2. **Detect.** Read each file and flag instances of the seven smells in [SMELLS.md](SMELLS.md).
   Judge by intent, not pattern-match — a `verb_noun` name is only a smell when it points
   at missing encapsulation, not every time. Prefer precision over volume; don't pad.
3. **Report.** Emit findings grouped by smell (see format below). Every finding needs a
   `file:line`, a one-line *why it smells*, and a short *refactor sketch* toward an object
   that owns the data.
4. **Summarize.** End with a smell-count table and the 3–5 highest-leverage refactors —
   usually a missing domain object that several findings all point at.

## Report format

````
## <Smell name>

**`path/to/file.ext:42`** — <one line: why this is a procedural smell>
  *Refactor:* <concrete OO move: which object should own this, what method replaces it>
````

Group findings under each smell heading. Sort smells by count, highest first.

## The seven smells

Brief list — full definitions, how-to-spot, and refactor patterns are in [SMELLS.md](SMELLS.md).

1. **verb_noun function names** — `parse_file`, `generate_report`: behavior that wants to live on an object.
2. **Temp-variable shuffling** — values pulled off an object's attributes, mutated, passed around.
3. **Primitive obsession** — raw hashes / anonymous structs passed across module boundaries.
4. **Grab-bag hash returns** — returning a hash of unrelated values to leak internal state to the caller.
5. **Multi-purpose / side-effecting functions** — does more than its name says.
6. **Call-site-driven signatures** — params/return shaped because *one* caller is known.
7. **(see SMELLS.md for the combined patterns and counter-examples)**

## Notes

- A smell is a *prompt to look closer*, not a verdict. Flag it, explain the cost, suggest the
  shape — let the reader decide.
- The recurring fix is the same: **find the data being shuffled, and the object that should own it.**
- Don't flag idiomatic free functions (pure helpers, stdlib-style utilities) just for being functions.
