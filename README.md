# Memory Keeper

A persistent, cross-agent memory system for code projects. Think of it as a durable `/compact` that survives across sessions and across different LLM tools — so if you were working with Claude yesterday and open Codex today, the next agent can read what was done and pick up cleanly.

## Features

- **Persistent Project Memory**: Record coding tasks, changes, and context that survive across sessions
- **Cross-Agent Compatible**: Works with Claude Code, Codex, Kimi, Gemini, and other LLM coding agents
- **Plain Markdown Format**: All memory files are standard markdown with YAML frontmatter — any LLM can read and write them
- **Automatic Task Recording**: Optionally record completed tasks including diffs, commands run, and follow-ups
- **Smart Context Retrieval**: Automatically load relevant prior work when starting a new task
- **Built-in Slash Commands**: `/read-memory` and `/update-memory` for easy access

## Quick Start

### Install User-Level (All Projects)

```bash
# macOS / Linux
mkdir -p ~/.claude/skills
cp -r memory-keeper ~/.claude/skills/

# Verify installation
ls ~/.claude/skills/memory-keeper/
# Output: SKILL.md, INSTALL.md, assets/
```

### Install Project-Level (Single Project)

```bash
# From your project root
mkdir -p .claude/skills
cp -r /path/to/memory-keeper .claude/skills/
```

### First Use

In Claude Code, type:

```
/read-memory
```

Or simply say: _"Let's start working — check if there's any project memory to inherit"_

On first use, the skill will detect that the `memory/` folder doesn't exist and offer to bootstrap it. Accept to create:

```
your-project/
├── memory/
│   ├── memory.md              # Master index of all tasks
│   ├── README.md              # Format specification for other agents
│   └── details/
│       └── YYYY-MM-DD_topic.md   # Full task records
└── .claude/
    └── commands/
        ├── read-memory.md     # Slash command
        └── update-memory.md   # Slash command
```

## How It Works

### Three Core Behaviors

#### 1. Bootstrap (First-Time Setup)
When you invoke `/read-memory` or `/update-memory` in a project without a `memory/` folder, the skill automatically creates the folder structure and installs slash commands.

#### 2. Record Tasks (After Completion)
After finishing a coding task, the skill asks if you want to record:
- **Topic**: Short kebab-case description (e.g., `auth-jwt-refactor`)
- **Status**: `complete` | `partial` | `failed` | `reverted`
- **Changes**: Human-readable file modifications with line ranges
- **Commands**: Shell commands executed with exit codes
- **Related Files**: All files examined or modified
- **Tags**: 1-4 lowercase labels (`auth`, `bugfix`, `perf`, etc.)
- **Follow-ups**: Known issues or next steps for future work

#### 3. Retrieve Context (New Task)
When starting a new task, the skill automatically:
1. Reads the master index (`memory.md`)
2. Analyzes your current task for relevant keywords
3. Loads 1-3 most relevant prior work entries
4. Summarizes inherited context before you start

### Memory File Structure

Each task is recorded in a detail file with this frontmatter:

```yaml
---
date: YYYY-MM-DD
topic: kebab-case-topic
agent: claude-opus-4.7
status: complete | partial | failed | reverted
tags: [tag1, tag2, tag3, tag4]
---
```

The master index (`memory.md`) maintains one-line summaries:
```
- [TYPE] Topic — agent-name — status-emoji — [details](details/FILENAME.md)
```

## Use Cases

✅ **Switch between LLM agents**: Seamlessly hand off work from Claude to Codex  
✅ **Long-running projects**: Pick up exactly where you left off days or weeks later  
✅ **Complex refactors**: Understand why previous decisions were made  
✅ **Debugging**: Review prior failed attempts to avoid repeating dead ends  
✅ **Multi-developer workflows**: Share context across different team members' sessions  

## Key Principles

- **Brutal honesty**: Failed attempts are recorded and valuable — prevents repeating mistakes
- **Agent identity matters**: Each entry records which LLM did the work
- **No bloat**: The index stays concise (one-line-per-entry); detailed info lives in `/details`
- **No secrets**: API keys, tokens, and credentials are automatically redacted
- **Plain markdown**: Future-proof — readable by any tool, no proprietary format
- **Append-only history**: Entries are never rewritten; corrections create new entries

## Slash Commands

After initialization, you'll have two commands available in your project:

### `/read-memory [keywords]`
Reads the project memory and loads relevant prior work. Optionally pass keywords to search for specific tasks.

```
/read-memory auth
/read-memory
```

### `/update-memory [topic-hint]`
Explicitly record the current task to memory. Useful when the automatic offer to save isn't enough.

```
/update-memory
/update-memory fix-login-bug
```

## Cross-Agent Interoperability

The `memory/` folder is designed to be read and written by **any LLM tool**. When you switch to Codex, Gemini, or another agent:

1. Point it to `memory/README.md` — contains the full format spec
2. Have it read `memory/memory.md` to get project overview
3. When finished, have it append new entries following the same format

No special tooling needed on the other agent's side.

## When to Record Tasks

✅ **Record**: Clear unit of work (code edits + tests, completed analysis, finished refactor)  
⏭️ **Skip**: Trivial one-liners (typos, formatting), pure Q&A sessions, ephemeral exploration

The skill asks before recording — you're always in control.

## Uninstall

```bash
# Remove the skill
rm -rf ~/.claude/skills/memory-keeper/
# or (project-level)
rm -rf .claude/skills/memory-keeper/

# Keep or delete the memory folder (optional)
rm -rf memory/  # If you want to remove project records
```

## Project Structure

```
memory-keeper/
├── SKILL.md                          # Main skill file for Claude
├── INSTALL.md                        # Installation instructions
├── README.md                         # This file
└── assets/
    ├── commands/
    │   ├── read-memory.md            # Installed as project slash command
    │   └── update-memory.md          # Installed as project slash command
    └── templates/
        ├── memory-readme.md          # Installed as memory/README.md
        ├── memory-index.md           # Installed as memory/memory.md (starter)
        └── detail-template.md        # Skeleton for task detail files
```



For issues, questions, or suggestions, please open an issue on GitHub or contact the maintainers.

---

**Memory Keeper**: Making your project work survive across sessions, tools, and agents. 🧠📝
