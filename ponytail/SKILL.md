---
name: ponytail
description: Personal cross-project working-style preferences - terse devspeak comments, preferring the simplest fix, one route per controller file, tests in their own file, scope discipline on diffs, caution and a test list when changing a widely used base component, an editable plan.md over the ExitPlanMode dialog, and sourcing claims - plus a growing list of code-quality checks (duplicated guards, ternary avoidance, return-await conventions, byte-size constants over bit-shifts, keeping failure behavior when swapping a data source, and more added over time). SQL rules live in the separate ponytail-sql skill. Load before writing comments/docstrings, adding tests, entering plan mode, reviewing a diff for style or quality issues, changing a shared base UI component, or writing/reviewing a controller/route handler.
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

## return await: match the file's existing convention

- `return await x(...)` vs `return x(...)`: same result, but `await` keeps this function's
  frame in the stack trace on rejection
- before dropping a "redundant" await, grep the file/repo for the same shape first -
  if it's already the convention there, match it instead of locally optimizing it away

## ternaries: avoid `cond ? a : b`

- prefer `if`/`else` over a ternary - reads slower, and unreadable fast once nested
- `.map()`/`.forEach()`/a plain `for` loop are all fine; the objection is the ternary
  operator, not the loop construct
- a fallback default (`a ?? b`) is fine as its own line; don't cram it into a bigger
  expression (e.g. inside an object literal alongside other fields)
- `flatMap((x) => cond ? [y] : [])` is a ternary doing a filter and a map at once; it also
  hides that items are being dropped. write the `if` out, and ask whether dropping is right
  (see failure behavior below)

## duplicated guards: one invariant, one place

when a diff adds a condition that decides *whether* something happens (a filter, an early
return, a `some()`/`any()` check), trace it forward to where the thing being gated is used -
if the callee already checks the same condition, that's one invariant enforced twice, and the
two will drift.

- reading each guard in isolation and asking "is this the simplest way to write *this line*"
  isn't enough - the question is whether this condition already exists somewhere else in the
  call chain
- a giveaway: a caller pre-filtering/pre-checking a list right before handing it to a function
  that immediately loops over it and filters/checks again
- fix by collapsing to one place, usually the callee (it's the one that actually needs the
  invariant to hold), and letting the caller call unconditionally

## swapping a data source: keep its failure behavior

when a change moves a read to a new source (db → s3, cache → api, sync job → direct fetch),
the happy path usually matches. what drifts is what happens when the source is missing,
empty or erroring.

- before calling it done, write down what the old and the new path each do for: missing,
  empty, erroring. compare them side by side
- the old path's fallback often lives somewhere else (a sync job that keeps the last good
  copy, a `WHERE count > 0` in a query); find it, don't assume
- the tell is a new branch that skips an item when its source isn't there. for a block/deny
  rule, silently leaving it out is fail-open
- if the old path kept serving the last good data, the new one must fail the request rather
  than return a quietly smaller result
- "same as the old X" is a claim; check it against the old code before saying it (see
  claims)

## readability: the 30-second test

for each changed or added file, ask: could someone unfamiliar with it follow this in about
30 seconds? soft signals, not hard fails - judgment still applies, especially for new files
(see below):
- a function running long (~40+ lines) without a clear reason to be one function
- conditionals nested 3+ levels deep
- a magic number/string where a named constant would explain itself
- a clever one-liner that trades clarity for cleverness
- an unclear variable name where a clearer one costs nothing
- a byte size written as a bit-shift (`8 << 20`) instead of `8 * 1024 * 1024`; keep
  `<<`/`>>` for real bitwise work (flags, masks)

new files get more benefit of the doubt than changes threaded into existing, working code:
a new module can carry more inherent complexity before any of the above is worth flagging,
since there's no existing structure or reader expectation it's disrupting.

## scope: don't let the diff explode

a task has an implicit boundary - the bug, the review comment, the feature asked for. stay
inside it.

- before wrapping up, look at the full diff/PR and ask whether every changed file/hunk
  actually serves the task - not "would this be a nice improvement while I'm here"
- drive-by refactors, unrelated renames, and opportunistic cleanup outside the task's
  files belong in a separate call-out to the user, not folded silently into the diff
- if a fix reveals a second, adjacent problem, surface it and ask rather than expanding
  the diff to cover it unasked

## base components: changes spread everywhere they're used

a base component is shared by many screens. one change to it changes every place that
uses it, including screens the task never touched.

- before editing a base component, confirm the change is intended for every usage, not
  just the screen you're working on
- the more places use it, the higher the bar; count the usages first (grep the component
  name) and say how many there are
- after any base component change, always write a list of what to test: every screen or
  component that uses it, grouped so it can actually be clicked through
- if that list is too big to realistically test, the change is probably the wrong one;
  don't make it
- look for an easier fix first:
  - a prop or variant that only the screen being fixed opts into
  - a local style or wrapper in the screen being fixed
  - a new component for the new case, leaving the base one as is
- only change the base component itself when every usage really should change, and say
  so explicitly in the PR/summary along with the test list

## tests: separate file, right-sized

never `#[cfg(test)] mod tests { ... }` inline in a Rust file; tests always live in their own
file. naming/wiring convention varies by repo - match the sibling test files already there
rather than inventing a new layout. same principle in other languages: prefer the project's
existing test-file convention over inlining test code next to implementation.

before writing new tests, look at how similar existing code is already tested (same kind of
test - unit/integration/e2e - and the same coverage style) and match that, rather than
inventing a new testing approach for this one change.

test *scope* should match the change, not sprawl past it:
- cover the new behavior and its realistic edge cases, not every theoretically possible
  input - exhaustiveness for its own sake is a smell, not a virtue
- don't add tests for pre-existing code the task didn't touch, and don't restructure the
  existing test suite as a side effect of adding new tests
- if the right-sized test count for a change feels like "a lot," that's usually a sign the
  change itself is doing more than the task asked - reconsider the change's scope first

## sql: load ponytail-sql

any time SQL comes into play (writing or reviewing a query, a repository function, a schema
file, or a controller/route handler that touches the database), also load the `ponytail-sql`
skill; its rules live there, not here.

## controllers: one route per file

a controller/route file registers exactly one endpoint. two handlers in one file isn't a
"related routes" convenience, it's a file you have to read in full to find the one handler
you came for.

- split on the route, not the resource - a collection endpoint and its `/:id` endpoint are
  two files even when they share a url prefix, a feature flag and a repository
- name each file after the registration function it exports, and register each separately
  wherever the app wires its routes up
- parsing/shaping the two would otherwise share goes in a `<feature>/<helper>.ts` beside
  them, not in whichever route file was written first
- a short guard both routes repeat (a feature-flag check, an auth precondition) is fine
  duplicated across the two files; don't add a wrapper to dedupe three lines

## validation: assert and return, not a declarative schema

for a request body, form, or any other narrow set of mostly-independent per-field checks, an
explicit assertion call per field reads faster than a declarative schema object describing the
whole thing at once - each line says exactly what it checks, and there's no schema DSL or its
edge cases (an `anyOf`-style validator silently coercing `null` to `0` while probing branches in
some validation-order it owns, not you) sitting between the code and what actually happens.

- `assert*` throws on failure and returns the validated, typed value; `is*` is a pure boolean
  predicate that never throws - don't blur the two into one function that sometimes throws and
  sometimes returns false
- extract the raw field, validate its shape, return it typed - that's the whole shape of the
  function. one line per field, at the top of the handler, reads like a list of "this field
  must be X" statements, top to bottom
- compose narrow checks instead of writing one broad one: assert the raw shape first (a
  non-empty string, an integer), then run a more specific check on the already-typed value (a
  real email or url check) - two small steps instead of one function trying to do both
- this earns its keep for narrow, mostly-independent per-field checks; a deeply cross-field or
  recursive shape is a different problem, and a real schema/parser library is the better fit
  there
- an `assert*`/`is*` helper nobody calls anymore is dead code like any other - don't keep
  speculative validators around "in case a future field needs them"; add the specific one when
  a real field needs it, same as any other YAGNI call

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
