---
name: memory-keeper
description: >
  Records and retrieves project modification history across different LLM agents
  and sessions, so Claude, Codex, Kimi, Gemini, and other coding agents can share
  progress on the same codebase. Use this skill whenever the user finishes a
  coding task, starts a new task on a project that has a memory/ folder, or invokes
  /read-memory or /update-memory. Also trigger on phrases like "continue from
  where we left off", "what's been done on this project", "record this change",
  "record this modification", "inherit previous progress", "read memory", or
  whenever you notice a memory/ folder in the project root and have not yet checked
  it this session. This is the persistent cross-agent memory that survives tool
  switches — do not rely on conversation compaction for this purpose.
---
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Memory Keeper

A durable, cross-agent memory system for code projects. Think of it as a `/compact` that survives across sessions and across LLM tools — so if the user was working with Claude yesterday and opens Codex today, the next agent can read what was done and pick up cleanly.

## What this skill owns

```text
<project-root>/
├── memory/
│   ├── memory.md            ← short master index, one line per task
│   ├── README.md            ← format spec any agent can read to understand the system
│   └── details/
│       └── YYYY-MM-DD_topic.md   ← full per-task records
└── .claude/
    └── commands/
        ├── read-memory.md   ← installed at bootstrap
        └── update-memory.md ← installed at bootstrap
```

The files inside `memory/` are plain markdown with YAML frontmatter. Any LLM can read and write them by following `memory/README.md` — no special tooling required. This skill just makes the workflow ergonomic for Claude Code.

---

## Three behaviors

### Behavior 1 — Bootstrap, first time in a project

**When:** You are about to read or write memory, or the user invokes `/read-memory` or `/update-memory`, but `memory/` does not exist.

**Steps:**

1. Ask the user, in their language:

   > "This project does not have memory-keeper set up yet. Initialize it? This creates a `memory/` folder and adds `/read-memory` and `/update-memory` slash commands to `.claude/commands/`."

2. If yes:

   ```bash
   mkdir -p memory/details .claude/commands
   ```

   * Copy `assets/templates/memory-readme.md` to `memory/README.md`
   * Copy `assets/templates/memory-index.md` to `memory/memory.md`
   * Copy `assets/commands/read-memory.md` to `.claude/commands/read-memory.md`
   * Copy `assets/commands/update-memory.md` to `.claude/commands/update-memory.md`

3. Confirm:

   > "Set up. `memory/` is ready. You can now use `/read-memory` and `/update-memory` anytime."

---

### Behavior 2 — Write memory, after task completion

**Trigger priority:**

1. The user invokes `/update-memory` — run this flow explicitly, with no confirmation needed because the request is already explicit.
2. The user says "record this", "save to memory", "log this", or "record this modification" — run this flow.
3. The user has just finished a clear unit of work, such as a code edit plus test run, completed analysis, or wrapped-up refactor, and is about to move on to something else — ask first before writing.

**IMPORTANT:** Do not auto-record without asking. The user chose "ask first" behavior. For borderline cases, ask:

> "Should I save this modification to memory?"

Offer: yes / yes with my edits / skip this one.

---

### Data to gather

Gather this methodically. Do not guess.

#### 1. Topic

Derive a short kebab-case phrase from the user's original prompt.

Examples:

* "Refactor JWT validation logic" → `auth-jwt-refactor`
* "Fix login timeout on Safari" → `fix-safari-login-timeout`
* "Research DB migration options" → `db-migration-research`

#### 2. Filename

Use:

```text
YYYY-MM-DD_{topic}.md
```

If that filename already exists, append `_v2`, `_v3`, and so on.

#### 3. Agent identity

Record which LLM did the work. Check your own system context.

Format:

```text
claude-<model-name>
```

Example:

```text
claude-opus-4.7
```

If you cannot determine the precise model, use:

```text
claude
```

#### 4. Status

Be honest. Use one of:

```text
complete | partial | failed | reverted
```

A failed attempt is more valuable to record than a success because it prevents the next agent from repeating the dead end. Never soften `failed` to `partial` to make things look better.

#### 5. Changes

Run git to get the actual diff data:

```bash
git diff HEAD              # modifications to tracked files
git status --short         # untracked/renamed/deleted
git diff --stat HEAD       # file-level summary
```

Translate the raw diff into a human-readable summary:

* File path
* Line ranges
* One-line description of what each hunk does

Do not paste raw diff output wholesale.

#### 6. Commands run

List the shell commands executed during the task, such as test, build, lint, or type-check commands, with their exit codes.

Include key output lines only: first 20 lines plus last 20 lines maximum. Never paste full logs.

Example:

```bash
$ npm test
# exit 0, 42 passed in 3.2s
```

#### 7. Related files

List files read or touched that inform this task's context. Include files examined even if they were not modified.

#### 8. Tags

Use 1 to 4 lowercase tags.

Common examples:

```text
auth, api, db, ui, perf, security, testing, refactor, bugfix, analysis
```

#### 9. Follow-ups / Known issues

Record anything the next agent should know, including landmines, deliberate TODOs, and things that were out of scope.

---

### Write the files

Create the detail file using `assets/templates/detail-template.md` as the skeleton, filled with the gathered data.

Prepend a one-line entry to `memory.md` under today's date section. Create the date heading if today's date does not exist yet.

Format:

```markdown
- [TYPE] Topic — agent-name — status-emoji status — [details](details/FILENAME.md)
```

`TYPE` must be one of:

```text
FEAT | BUG | REFACTOR | ANALYSIS | RESEARCH | DOCS
```

Status emoji mapping:

```text
✅ complete
⚠️ partial
❌ failed
↩️ reverted
```

Confirm to the user with a brief summary of what was recorded and the path to the detail file.

---

### Behavior 3 — Read memory, new task or `/read-memory`

**Trigger:**

* The user invokes `/read-memory`, optionally with keyword arguments.
* A fresh session starts and the project has `memory/` — read proactively before taking on the first substantive task.
* The user says "continue from where we left off", "what's been done", "inherit previous progress", or "read memory".

---

### Workflow

#### 1. Always read `memory/memory.md` first

It is cheap and gives you the full project overview plus the "Conventions learned about this project" section. That section is especially valuable.

#### 2. Parse the current task

Parse the current task from `$ARGUMENTS` if `/read-memory` was invoked with arguments. Otherwise, parse it from the user's current prompt.

Look for:

* Domain keywords, such as auth, payment, cache
* File paths or module names mentioned
* Task-type hints, such as bugfix, refactor, or analysis

#### 3. Search `memory/details/` for relevant entries

Example search strategy, adapted per query:

```bash
grep -ril "<keyword>" memory/details/
grep -rl "related_files:.*<file-path>" memory/details/
```

Also scan `tags:` in frontmatter.

Rank candidates by:

1. Number of keyword or tag hits
2. Recency

#### 4. Load the 1 to 3 most relevant detail files

Do not load everything — context budget matters. If more than 3 entries look relevant, pick the top 3 and mention that the others exist.

#### 5. Summarize the inherited context to the user

Include:

* What prior work is relevant, with links to the detail files
* Any flagged follow-ups or known issues from those prior tasks
* Any entries where `agent:` differs from your own model — these are cross-agent handoffs and deserve a light verification pass before trusting

#### 6. Proceed with the user's task

Continue with the task using the inherited context.

If no relevant details are found, say explicitly:

> "Checked memory — nothing relevant to this task. Starting fresh."

Do not invent relevance that is not there.

---

## Key principles

### Brutally honest status

A recorded failure is worth more than a recorded false success. If something did not work, say so.

### Agent identity matters

Always record `agent:` in frontmatter. When a human switches from Claude to Codex, or vice versa, the next agent needs to know which pieces of the history it can trust as familiar terrain versus foreign work worth verifying.

### Do not bloat `memory.md`

The index stays one-line-per-entry. Deep details live in `details/`.

If `memory.md` grows past roughly 200 entries, offer to archive older ones to:

```text
memory/archive/YYYY-QN.md
```

### No secrets

Never record API keys, tokens, passwords, connection strings with credentials, or PII from test data.

If command output contains such values, redact them before writing:

```text
[REDACTED]
```

### Git diff is source material, not the output

Run `git diff` to get ground truth about what changed, but transcribe it into readable prose — file path, line range, and what the change does.

Nobody wants to read raw unified diffs inside memory files.

### Respect the user's language

Detect which language the user is working in from their prompts and ask confirmation questions in that language.

Memory file content stays in English for cross-agent interoperability, because other LLMs may read it.

### Preserve history

Do not rewrite existing entries.

If a prior entry turns out to be wrong, add a new entry that corrects it, with a reference back. The log is append-only.

---

## When not to use this skill

* Trivial one-liners, such as a typo fix, formatting pass, or variable rename — not worth an entry
* Pure Q&A sessions where no code is modified
* When the user explicitly says "do not record this", "ephemeral session", or "just testing"
* Speculative exploratory edits that the user immediately reverted and does not care about

Use judgment. When in doubt, ask.

---

## Slash commands this skill installs

After bootstrap, the project will have:

* `/read-memory [optional: keywords]` — runs Behavior 3
* `/update-memory [optional: topic hint]` — runs Behavior 2 explicitly

If the user invokes one of these and the corresponding file is missing, such as if they deleted `.claude/commands/` or cloned a repo with `memory/` but no commands, offer to re-run bootstrap for just the commands.

---

## Cross-agent interop notes

The files this skill creates are standard markdown. Any LLM coding agent can read and write them without having this skill installed.

The key is `memory/README.md`. It is written as instructions for whichever agent is reading it, explaining the format.

When you, Claude, take over work from a different agent:

* Trust but verify — run tests, skim the diffs, and do not assume their "complete" is truly complete
* Their `Follow-ups / Known Issues` sections are the highest-signal part of their entry
* If you notice a prior entry is wrong, add a correction entry rather than editing the original

---

## Reference files

* `assets/templates/detail-template.md` — skeleton for detail files
* `assets/templates/memory-index.md` — starter content for `memory.md`
* `assets/templates/memory-readme.md` — the README installed to `memory/README.md`
* `assets/commands/read-memory.md` — the slash command installed to `.claude/commands/read-memory.md`
* `assets/commands/update-memory.md` — the slash command installed to `.claude/commands/update-memory.md`
