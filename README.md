# skills

Personal Claude Code skills, portable across every project (not tied to one repo's
`.claude/skills`).

## Install

Install every skill in this repo with the [skills CLI](https://github.com/vercel-labs/skills):

```sh
npx skills add bitterpanda63/skills
```

Or just one:

```sh
npx skills add bitterpanda63/skills --skill ponytail
```

Add `-g` to install globally (all projects) instead of into the current one.

## ponytail

Cross-project working-style preferences: terse devspeak comments, prefer the simplest fix,
keep PR/diff scope from creeping, right-sized tests, tests in their own file, plans as an
editable `plan.md`, and sourcing claims about undocumented behavior instead of stating
guesses as fact. See [`ponytail/SKILL.md`](ponytail/SKILL.md).

Invoke it in Claude Code with `/ponytail`, or let it load on its own when you're writing
comments, adding tests, entering plan mode or reviewing a diff.
