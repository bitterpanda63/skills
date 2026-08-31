---
name: ponytail
description: Personal cross-project working-style preferences - terse devspeak comments that usually run ~1-2 lines, preferring the simplest fix that works, keeping a PR/diff's scope from creeping beyond the task, right-sizing test coverage to the change, tests in their own file (never inline), plans written as an editable plan.md instead of the ExitPlanMode dialog, and flagging unsourced inferences as guesses rather than facts. Load before writing comments/docstrings, adding tests, entering plan mode, stating how undocumented/internal behavior works, or wrapping up a PR/diff.
---

# ponytail

personal style rules that apply in every repo, not just one project's CLAUDE.md. project-specific
CLAUDE.md rules still win where they're more specific than this.

## comments: devspeak, usually short

barely any comments; most code is self-explanatory.

when one's warranted (non-obvious constraint/trade-off/reason, never restating the code):
- not a full sentence; no unneeded capitalization on short comments
- no em-dashes; use `;` to join two clauses
- one line, short, dense - multiple ideas as consecutive one-liners, not prose paragraphs
- **default to ~1-2 lines of real substance.** that's the common case, not a hard limit -
  a genuinely load-bearing explanation (a subtle invariant, a why that needs the full
  causal chain) can run longer. the smell to watch for is a comment growing because the
  reasoning felt incomplete, not because the content actually needed the extra lines; when
  in doubt, try cutting to the one fact that matters or splitting into separate one-liners
  first, and only keep the longer version if that trim actually lost something
- if a function needs a comment to be understood, split it up and name the pieces instead
- don't describe the old way something was done unless that history is load-bearing
- when reviewing a diff (own or someone else's), flag comment blocks that look long relative
  to what they're saying as trim candidates, same as any other cleanup finding

## simplicity: take the simpler path

when two approaches both solve the task, prefer the one with less code, fewer moving parts,
or fewer new abstractions - even if the other felt more thorough or "correct" while writing it.

- before calling something done, ask "is there a simpler way to do this" once, concretely
  (not rhetorically) - if yes, do that instead
- prefer reusing/extending something that already exists over adding a parallel mechanism
- a bug fix doesn't need surrounding cleanup; a one-shot operation doesn't need a helper;
  don't design for hypothetical future requirements (see global CLAUDE.md's "no beyond-task
  abstractions" rule - this is the same principle, applied as an explicit check step)

## scope: don't let the diff explode

a task has an implicit boundary - the bug, the review comment, the feature asked for. stay
inside it.

- before wrapping up, look at the full diff/PR and ask whether every changed file/hunk
  actually serves the task - not "would this be a nice improvement while I'm here"
- drive-by refactors, unrelated renames, and opportunistic cleanup outside the task's
  files belong in a separate call-out to the user, not folded silently into the diff
- if a fix reveals a second, adjacent problem, surface it and ask rather than expanding
  the diff to cover it unasked

## tests: separate file, right-sized

never `#[cfg(test)] mod tests { ... }` inline in a Rust file; tests always live in their own
file. naming/wiring convention varies by repo - match the sibling test files already there
rather than inventing a new layout. same principle in other languages: prefer the project's
existing test-file convention over inlining test code next to implementation.

test *scope* should match the change, not sprawl past it:
- cover the new behavior and its realistic edge cases, not every theoretically possible
  input - exhaustiveness for its own sake is a smell, not a virtue
- don't add tests for pre-existing code the task didn't touch, and don't restructure the
  existing test suite as a side effect of adding new tests
- if the right-sized test count for a change feels like "a lot," that's usually a sign the
  change itself is doing more than the task asked - reconsider the change's scope first

## plans: editable plan.md, not the approval dialog

when a task warrants a written plan, write it to a real `plan.md` (or similar) file in the
repo via Write/Edit, not just the internal plan-mode file. skip/decline the ExitPlanMode
approval flow; the file itself is the deliverable, reviewable and editable in the user's own
IDE. still fine to use plan mode's exploration phase internally - just don't exit through it.

## claims: don't dress up inference as fact

don't state inferred/undocumented behavior (OS internals, library internals, anything the
docs don't spell out) as settled fact. naming a specific internal component or mechanism
doesn't make a guess into a fact, it just makes it sound like one.
- quote the primary source for each link in a causal chain
- say out loud which link is unsourced
- if a claim is load-bearing for a conclusion, either source it or propose the empirical test
  that would settle it
