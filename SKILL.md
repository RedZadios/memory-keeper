---
name: memory-keeper
description: Records and retrieves project modification history across different LLM agents and sessions, so Claude, Codex, Kimi, Gemini, and other coding agents can share progress on the same codebase. Use this skill whenever the user finishes a coding task (to offer recording the change), starts a new task on a project that has a memory/ folder (to inherit relevant prior context before working), or invokes /read-memory or /update-memory. Also trigger on phrases like "continue from where we left off", "what's been done on this project", "record this change", "記錄這次修改", "繼承之前的進度", "讀一下 memory", or whenever you notice a memory/ folder in the project root and haven't yet checked it this session. This is the persistent cross-agent memory that survives tool switches — do not rely on conversation compaction for this purpose.
---

# Memory Keeper

A durable, cross-agent memory system for code projects. Think of it as a `/compact` that survives across sessions and across LLM tools — so if the user was working with Claude yesterday and opens Codex today, the next agent can read what was done and pick up cleanly.

## What this skill owns

```
<project-root>/
├── memory/
│   ├── memory.md            ← short master index (one line per task)
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

### Behavior 1 — Bootstrap (first time in a project)

**When:** You're about to read/write memory, or the user invokes `/read-memory` or `/update-memory`, but `memory/` doesn't exist.

**Steps:**
1. Ask the user (in their language):
   > "This project doesn't have memory-keeper set up yet. Initialize it? This creates a `memory/` folder and adds `/read-memory` + `/update-memory` slash commands to `.claude/commands/`."
2. If yes:
   - `mkdir -p memory/details .claude/commands`
   - Copy `assets/templates/memory-readme.md` → `memory/README.md`
   - Copy `assets/templates/memory-index.md` → `memory/memory.md`
   - Copy `assets/commands/read-memory.md` → `.claude/commands/read-memory.md`
   - Copy `assets/commands/update-memory.md` → `.claude/commands/update-memory.md`
3. Confirm: `"Set up. memory/ ready. You can now use /read-memory and /update-memory anytime."`

### Behavior 2 — Write memory (after task completion)

**Trigger priority:**
1. User invokes `/update-memory` — run this flow explicitly, no confirmation needed (already explicit).
2. User says "record this" / "save to memory" / "log this" / "記錄一下" — run this flow.
3. User just finished a clear unit of work (code edit + test run, completed analysis, wrapped up a refactor) and is about to move on to something else — **ask first** before writing.

**IMPORTANT:** Do NOT auto-record without asking. The user chose "ask first" behavior. For borderline cases, ask: `"要把這次修改記到 memory 嗎?"` (or equivalent in their language), offering: yes / yes with my edits / skip this one.

**Data to gather** (do this methodically — don't guess):

1. **Topic** — derive a short kebab-case phrase from the user's original prompt. Examples:
   - "重構 JWT 驗證邏輯" → `auth-jwt-refactor`
   - "Fix login timeout on Safari" → `fix-safari-login-timeout`
   - "研究 db migration 方案" → `db-migration-research`

2. **Filename** — `YYYY-MM-DD_{topic}.md`. If that filename already exists, append `_v2`, `_v3`, etc.

3. **Agent identity** — record which LLM did the work. Check your own system context. Format: `claude-<model-name>` (e.g. `claude-opus-4.7`). If you can't determine precisely, use `claude`.

4. **Status** — be honest: `complete` | `partial` | `failed` | `reverted`. A failed attempt is **more** valuable to record than a success, because it prevents the next agent from repeating the dead end. Never soften "failed" to "partial" to make things look better.

5. **Changes** — run git to get actual diff data:
   ```bash
   git diff HEAD              # modifications to tracked files
   git status --short         # untracked/renamed/deleted
   git diff --stat HEAD       # file-level summary
   ```
   Translate the raw diff into a human-readable summary: file path, line ranges, and one-line description of what each hunk does. Don't paste raw diff output wholesale.

6. **Commands run** — list the shell commands executed during the task (test, build, lint, type-check, etc.) with their exit codes. Include key output lines (first 20 + last 20 lines max; never paste full logs). Example:
   ```bash
   $ npm test
   # exit 0, 42 passed in 3.2s
   ```

7. **Related files** — files read or touched that inform this task's context (include files examined even if not modified).

8. **Tags** — 1 to 4 lowercase tags. Common ones: `auth`, `api`, `db`, `ui`, `perf`, `security`, `testing`, `refactor`, `bugfix`, `analysis`.

9. **Follow-ups / Known issues** — anything the next agent should know. Landmines, deliberate TODOs, things that were out of scope.

**Write the files:**
- Create the detail file using `assets/templates/detail-template.md` as the skeleton, filled with the gathered data.
- Prepend a one-line entry to `memory.md` under today's date section (create the date heading if today doesn't have one yet). Format:
  ```
  - [TYPE] Topic — agent-name — status-emoji status — [details](details/FILENAME.md)
  ```
  TYPE ∈ `FEAT` | `BUG` | `REFACTOR` | `ANALYSIS` | `RESEARCH` | `DOCS`
  status emoji: ✅ complete, ⚠️ partial, ❌ failed, ↩️ reverted

**Confirm** to the user with a brief summary of what was recorded and the path to the detail file.

### Behavior 3 — Read memory (new task / `/read-memory`)

**Trigger:**
- User invokes `/read-memory` (optionally with keyword arguments)
- Fresh session and the project has `memory/` — read proactively before taking on the first substantive task
- User says "continue from where we left off" / "what's been done" / "繼承之前的進度" / "讀一下 memory"

**Workflow:**

1. **Always read `memory/memory.md` first.** It's cheap and gives you the full project overview plus the "Conventions learned about this project" section — that one is especially valuable.

2. **Parse the current task** (from `$ARGUMENTS` if `/read-memory` was invoked with args, otherwise from the user's current prompt) for:
   - Domain keywords (auth, payment, cache, etc.)
   - File paths or module names mentioned
   - Task-type hints (bugfix, refactor, analysis…)

3. **Search `memory/details/`** for relevant entries:
   ```bash
   # Example search strategy (adapt per query):
   grep -ril "<keyword>" memory/details/
   grep -rl "related_files:.*<file-path>" memory/details/
   ```
   Also scan `tags:` in frontmatters. Rank candidates by: (a) number of keyword/tag hits, (b) recency.

4. **Load 1–3 most relevant detail files.** Do NOT load everything — context budget matters. If >3 look relevant, pick the top 3 and mention the others exist.

5. **Summarize the inherited context** to the user:
   - What prior work is relevant (with links to the detail files)
   - Any flagged follow-ups or known issues from those prior tasks
   - Flag any entries where `agent:` differs from your own model — those are cross-agent handoffs and deserve a light verification pass before trusting
6. **Then proceed with the user's task**, armed with this context.

If no relevant details found, say so explicitly: `"Checked memory — nothing relevant to this task. Starting fresh."` Don't invent relevance that isn't there.

---

## Key principles

**Brutally honest status.** A recorded failure is worth more than a recorded false success. If something didn't work, say so.

**Agent identity matters.** Always record `agent:` in frontmatter. When a human switches from Claude to Codex (or vice versa), the next agent needs to know which pieces of the history it can trust as "familiar terrain" vs "foreign work worth verifying."

**Don't bloat `memory.md`.** The index stays one-line-per-entry. Deep stuff lives in `details/`. If `memory.md` grows past ~200 entries, offer to archive older ones to `memory/archive/YYYY-QN.md`.

**No secrets.** Never record API keys, tokens, passwords, connection strings with credentials, or PII from test data. If a command output contains such values, redact them before writing (`[REDACTED]`).

**Git diff is source material, not the output.** Run `git diff` to get ground truth about what changed, but transcribe it into readable prose — file path, line range, and what the change does. Nobody wants to read raw unified diffs inside memory files.

**Respect the user's language.** Detect which language the user is working in (from their prompts) and ask confirmation questions in that language. Memory file CONTENT stays in English for cross-agent interoperability (other LLMs may read it).

**Preserve history.** Don't rewrite existing entries. If a prior entry turned out to be wrong, add a new entry that corrects it (with a reference back). The log is append-only.

---

## When NOT to use this skill

- Trivial one-liners (typo fix, formatting pass, renaming a variable) — not worth an entry
- Pure Q&A sessions where no code is modified
- When the user explicitly says "don't record this" / "ephemeral session" / "just testing"
- Speculative exploratory edits the user immediately reverted and doesn't care about

Use judgment. If in doubt, ask.

---

## Slash commands this skill installs

After bootstrap, the project will have:

- `/read-memory [optional: keywords]` — runs Behavior 3
- `/update-memory [optional: topic hint]` — runs Behavior 2 explicitly

If the user invokes one of these and the corresponding file is missing (e.g., they deleted `.claude/commands/` or cloned a repo with `memory/` but no commands), offer to re-run bootstrap for just the commands.

---

## Cross-agent interop notes

The files this skill creates are standard markdown. Any LLM coding agent can read and write them without having this skill installed. The key is `memory/README.md` — it's written as instructions for whichever agent is reading it, explaining the format.

When you (Claude) take over work from a different agent:
- Trust but verify — run tests, skim the diffs, don't assume their "complete" is truly complete
- Their `Follow-ups / Known Issues` sections are the highest-signal part of their entry
- If you notice a prior entry is wrong, add a correction entry rather than editing the original

---

## Reference files

- `assets/templates/detail-template.md` — skeleton for detail files
- `assets/templates/memory-index.md` — starter content for `memory.md`
- `assets/templates/memory-readme.md` — the README installed to `memory/README.md`
- `assets/commands/read-memory.md` — the slash command installed to `.claude/commands/read-memory.md`
- `assets/commands/update-memory.md` — the slash command installed to `.claude/commands/update-memory.md`
