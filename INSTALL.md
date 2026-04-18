# Memory Keeper — Installation

A Claude Code skill that gives your projects a cross-agent, cross-session memory.

## What it does

- After you finish a coding task, asks if you want to record the modification + results to a persistent project memory
- At the start of a new task, selectively loads relevant prior work so you don't start from scratch
- The files it creates are plain markdown — **other LLM tools (Codex, Kimi, Gemini, etc.) can read and write them too**, so context survives tool switches
- Provides `/read-memory` and `/update-memory` slash commands in each project where it's used

## Install (user-level — available in all projects)

```bash
# macOS / Linux
mkdir -p ~/.claude/skills
cp -r memory-keeper ~/.claude/skills/

# Verify
ls ~/.claude/skills/memory-keeper/
# should show: SKILL.md, INSTALL.md, assets/
```

## Install (project-level — only this project)

```bash
# From your project root
mkdir -p .claude/skills
cp -r /path/to/memory-keeper .claude/skills/
```

## First use

Open Claude Code in any project and type:

```
/read-memory
```

Or just say something like _"let's start working — check if there's any project memory to inherit"_.

On first use, the skill will detect that the `memory/` folder doesn't exist and offer to bootstrap it. Accept, and it'll create:

```
your-project/
├── memory/
│   ├── memory.md
│   ├── README.md
│   └── details/
└── .claude/
    └── commands/
        ├── read-memory.md
        └── update-memory.md
```

From then on, `/read-memory` and `/update-memory` are available in that project, and the skill will proactively offer to record memory when you wrap up a task.

## Using from other LLM tools

The `memory/` folder is designed to be read/written by ANY LLM, not just Claude. When you switch to Codex, Kimi, Gemini, Cursor, or anything else:

1. Point the agent at `memory/README.md` — it contains the full format spec
2. Tell it to read `memory/memory.md` to get project state
3. When it finishes work, it can append to `memory.md` and create a new file in `memory/details/` following the same format

No special tooling needed on the other agent's side — just plain markdown files.

## Uninstall

- Remove `~/.claude/skills/memory-keeper/` (or `.claude/skills/memory-keeper/` if project-level)
- The `memory/` folder and `.claude/commands/` files in your project remain — they're valuable records. Delete manually if you want to remove them.

## Structure of this skill

```
memory-keeper/
├── SKILL.md                          ← main skill file (Claude reads this)
├── INSTALL.md                        ← this file
└── assets/
    ├── commands/
    │   ├── read-memory.md            ← installed to project's .claude/commands/
    │   └── update-memory.md          ← installed to project's .claude/commands/
    └── templates/
        ├── memory-readme.md          ← installed to project's memory/README.md
        ├── memory-index.md           ← installed to project's memory/memory.md
        └── detail-template.md        ← used as skeleton for each detail file
```
