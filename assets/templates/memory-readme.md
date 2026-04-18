# `memory/` — Cross-Agent Project Memory

This folder is a shared workspace memory for LLM coding agents (Claude, Codex, Kimi, Gemini, Cursor, and others) working on this project. **Any agent can read and write to it** — this file is the complete spec. No special tooling required; it's all plain markdown.

The goal: when the human switches agents or starts a new session, the incoming agent can pick up cleanly without redoing analysis or repeating dead-end attempts.

---

## Structure

```
memory/
├── memory.md        ← short master index (one line per task, reverse chronological)
├── README.md        ← this file
└── details/
    └── YYYY-MM-DD_topic.md   ← one file per task with full details
```

---

## How to READ (at the start of a new task)

1. **Always read `memory.md` first.** It's lightweight and gives you the full project overview plus the "Conventions learned about this project" section — that section is often the highest-signal part.

2. **Parse the current task** for keywords, mentioned file paths, and task type (bugfix/refactor/feature/etc).

3. **Search `details/` for relevant entries** using the keywords + `tags:` + `related_files:` in each file's frontmatter. Don't load everything — only the 1–3 most relevant.

4. **Pay attention to the `agent:` field.** If a prior entry was written by a different LLM than you, treat its claims as "needs light verification" rather than "already true." Different agents have different blind spots.

5. **Especially read `Follow-ups / Known Issues` sections** of the relevant prior entries — that's where landmines are flagged.

---

## How to WRITE (at the end of a task)

1. Create `details/YYYY-MM-DD_{kebab-case-topic}.md`. If today already has a file with that topic, suffix `_v2`, `_v3`, etc.

2. Fill in the detail file using the template below.

3. Prepend a one-line entry to `memory.md` under today's date heading (create the heading if today doesn't have one yet).

4. **Be honest about status.** A recorded failure is more valuable than a misleading "complete" — it prevents the next agent from repeating the same dead end.

---

## Detail file template

```markdown
---
date: YYYY-MM-DD
agent: <llm-identifier>            # e.g. claude-opus-4.7, codex, kimi-k2, gemini-2.5-pro
task_type: <type>                  # feature | bugfix | refactor | analysis | research | docs
status: <status>                   # complete | partial | failed | reverted
tags: [tag1, tag2]                 # 1-4 lowercase tags
related_files:
  - path/to/file1
  - path/to/file2
prompt: "<user's original prompt>"
---

## Summary
One or two sentences: what was attempted, what the outcome was.

## Analysis / Approach
(Only if there was genuine investigation. Skip for mechanical changes.)

## Changes
### `path/to/file1`
- L23-L45: what this hunk does

### `path/to/file2`
- L12-L30: what this hunk does

## Commands Run
\`\`\`bash
$ npm test
# exit: 0, 42 passed
\`\`\`

## Results
- ✅ Tests pass
- ⚠️ Lint has 2 warnings (not related to this change)

## Follow-ups / Known Issues
- Refresh token currently in localStorage — should move to httpOnly cookie
- Rate limiting not yet implemented
```

---

## `memory.md` log line format

```
- [TYPE] Topic summary — agent-name — status — [details](details/FILENAME.md)
```

Where:
- **TYPE** ∈ `FEAT` | `BUG` | `REFACTOR` | `ANALYSIS` | `RESEARCH` | `DOCS`
- **status emoji**: ✅ complete, ⚠️ partial, ❌ failed, ↩️ reverted

Example:

```
- [BUG] Fix login timeout on Safari — claude-opus-4.7 — ✅ complete — [details](details/2026-04-18_fix-safari-login-timeout.md)
- [REFACTOR] JWT auth with refresh tokens — codex — ⚠️ partial (e2e not done) — [details](details/2026-04-18_auth-jwt-refactor.md)
- [ANALYSIS] DB migration strategy options — claude-sonnet-4.6 — ✅ complete — [details](details/2026-04-17_db-migration-analysis.md)
```

---

## Never record

- API keys, tokens, passwords, secrets
- Connection strings with embedded credentials
- User PII from test/production data
- Internal infrastructure details that shouldn't leak

If a command output contains such values, replace with `[REDACTED]` before writing.

---

## Cross-agent handoff etiquette

- **Don't rewrite existing entries.** The log is append-only. If a prior entry turns out to be wrong, add a new entry that corrects it (with a reference back to the original).
- **Verify claims from other agents before trusting them.** Especially for entries marked `complete` by a different LLM — run the tests yourself, skim the diffs, confirm the change is actually in place.
- **Update "Conventions learned" in `memory.md`** when you discover a project pattern that a future agent should know. Keep entries terse — one bullet per convention.
- **Flag landmines you find but can't fix** in your entry's `Follow-ups / Known Issues`. The next agent will thank you.

---

## Archival

If `memory.md` grows past ~200 entries, older entries can be moved to `memory/archive/YYYY-QN.md` (quarterly archive files) to keep the main log readable. Keep the last ~3 months live in `memory.md`.
