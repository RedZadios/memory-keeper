---
description: Record the current task's modifications to project memory
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(git diff*), Bash(git status*), Bash(git log*), Bash(ls *), Bash(mkdir *)
---

# /update-memory

Record the work just completed in this session to the project's memory system.

**Optional argument**: $ARGUMENTS — a topic hint for the filename. If empty, infer from the user's original prompt.

## Workflow

1. **Check `memory/` exists.** If not, offer to initialize via the `memory-keeper` skill first.

2. **Determine the topic** (for the filename):
   - From `$ARGUMENTS` if provided, OR
   - Infer from the user's original task prompt this session (short kebab-case, e.g. `auth-jwt-refactor`)

3. **Run git to detect actual changes:**
   ```bash
   git diff HEAD              # modifications to tracked files
   git status --short         # untracked/renamed/deleted files
   git diff --stat HEAD       # file-level summary
   ```

4. **Gather the data** (see `memory/README.md` for the full schema):
   - **Agent**: the LLM running this — record the model identifier in frontmatter (e.g. `claude-opus-4.7`)
   - **Status**: `complete` | `partial` | `failed` | `reverted` — **be honest**. If ambiguous, ask the user. Never soften "failed" to "partial" to look better.
   - **Changes**: translate git diff into readable form — file path, line range, brief description per hunk. Don't paste raw diff wholesale.
   - **Commands run**: shell commands executed during the task (tests, builds, lints) with exit codes and key output lines (first 20 + last 20 max; never paste full logs)
   - **Related files**: files read or modified
   - **Tags**: 1-4 lowercase tags (auth, perf, refactor, bugfix, etc.)
   - **Follow-ups / Known Issues**: landmines, TODOs, out-of-scope items the next agent should know

5. **Redact secrets** — no API keys, tokens, passwords, or credentials in command output. Replace with `[REDACTED]`.

6. **Write `memory/details/YYYY-MM-DD_{topic}.md`** using the template in `memory/README.md`. If a file with that date+topic already exists, suffix `_v2`, `_v3`, etc.

7. **Prepend a one-line entry to `memory/memory.md`** under today's date heading (create the heading if missing). Format:
   ```
   - [TYPE] Topic — agent-name — status-emoji status — [details](details/FILENAME.md)
   ```
   TYPE: `FEAT` | `BUG` | `REFACTOR` | `ANALYSIS` | `RESEARCH` | `DOCS`
   Status emoji: ✅ complete, ⚠️ partial, ❌ failed, ↩️ reverted

8. **If you learned a project convention** during this task (a pattern the next agent should know), add a bullet to the "Conventions learned about this project" section in `memory.md`. Keep it terse.

9. **Confirm** with a short summary: what was written, the path to the detail file, and the one-line log entry added.

## Example invocations

- `/update-memory` — infer topic from session context
- `/update-memory auth-refactor` — explicit topic hint
- `/update-memory "fix timeout bug"` — multi-word hint (will be kebab-cased)
