---
date: YYYY-MM-DD
agent: <llm-identifier>            # e.g. claude-opus-4.7, codex, kimi-k2, gemini-2.5-pro
task_type: <type>                  # feature | bugfix | refactor | analysis | research | docs
status: <status>                   # complete | partial | failed | reverted
tags: [tag1, tag2]                 # 1-4 lowercase tags
related_files:                     # files read or modified
  - path/to/file1
  - path/to/file2
prompt: "<user's original prompt, verbatim or close paraphrase>"
---

## Summary

One or two sentences: what was attempted, what the outcome was.

## Analysis / Approach

(Include only if there was genuine investigation, design thinking, or research.
For pure mechanical edits, delete this section entirely.)

- What was investigated
- What options were considered
- Why the chosen approach was selected

## Changes

### `path/to/file1`

- L<start>-L<end>: Brief description of what this hunk does
- L<line>: Brief description of what changed

### `path/to/file2`

- L<start>-L<end>: Brief description

(Repeat per file. Keep descriptions short — the git history has the exact diff.)

## Commands Run

```bash
$ <command>
# exit: 0
# key output: <first relevant line or two>
```

```bash
$ <another command>
# exit: 1
# error: <the relevant error snippet>
```

## Results

- ✅ / ⚠️ / ❌ Test suite: <name> — <outcome>
- ✅ / ⚠️ / ❌ Build / lint / type-check — <outcome>
- Any other verification step

## Follow-ups / Known Issues

- Anything the next agent should know
- Deliberate TODOs left in the code
- Things that were out of scope
- Landmines discovered but not fixed
