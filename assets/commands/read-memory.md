---
description: Load relevant prior work from memory/ folder at the start of a new task
allowed-tools: Read, Glob, Grep, Bash(ls *), Bash(cat *), Bash(git log *)
---

# /read-memory

Load relevant prior work from this project's `memory/` folder to inherit context before starting a new task.

**Optional argument**: $ARGUMENTS — if provided, use as the primary search query. If empty, infer search terms from the user's current task context.

## Workflow

1. **Check `memory/` exists.** If not, tell the user and offer to initialize it (invoke the `memory-keeper` skill to bootstrap).

2. **Read `memory/memory.md` fully.** It's intentionally short. Pay attention to:
   - The "Conventions learned about this project" section (valuable regardless of task)
   - Recent entries in the log

3. **Determine search terms**:
   - If `$ARGUMENTS` provided → use those keywords
   - Otherwise → extract keywords from the user's current task prompt (domain terms, file names, feature names, task type)

4. **Search `memory/details/`** for relevant entries:
   - `grep -ril "<keyword>" memory/details/` for body/frontmatter matches
   - Scan `tags:` and `related_files:` fields in each file's frontmatter
   - Rank by relevance × recency

5. **Read the 1–3 most relevant detail files.** Don't load everything — context budget matters. If more than 3 look relevant, pick top 3 and briefly note the others exist.

6. **Summarize for the user**:
   - What prior context is being inherited (with links to the detail files)
   - Any flagged `Follow-ups / Known Issues` from prior work
   - **Flag cross-agent entries**: if any relevant entry's `agent:` differs from the current agent, note it — those claims may warrant verification.

7. If nothing relevant found, say so honestly: "Checked memory — no prior work on this topic. Starting fresh." Don't invent relevance.

8. **Then proceed with the user's task**, armed with the inherited context.

## Example invocations

- `/read-memory` — infer search from current context
- `/read-memory auth` — search for anything related to authentication
- `/read-memory payment refund flow` — multi-keyword search
